---
type: resource-pattern
components: []
deployment_types: [helm]
resource_types: [security-context]
architecture: []
source_examples:
  - blueprint: "video-search-and-summarization"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-video-search-and-summarization"
    notes: "Demonstrates custom SCC for running containers with runAsUser: 0 and NIM model PVCs"
    approach: "A"
  - blueprint: "generative-virtual-screening"
    source_repo: "https://github.com/NVIDIA-BioNeMo-blueprints/generative-virtual-screening"
    fork_repo: "https://github.com/rh-ai-quickstart/generative-virtual-screening"
    notes: "Custom SCC for BioNeMo NIMs with runAsUser: 0 for MSA init container and SELinux skip for large model PVCs (1.5TB)"
    approach: "A"
  - blueprint: "generative-protein-binder-design"
    source_repo: "https://github.com/NVIDIA-BioNeMo-blueprints/generative-protein-binder-design"
    fork_repo: "https://github.com/rh-ai-quickstart/generative-protein-binder-design"
    notes: "Custom SCC with seLinuxContext: RunAsAny for AlphaFold2 2TB PVCs with millions of genomic database files to prevent recursive relabeling timeouts"
    approach: "A"
  - blueprint: "rag"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/rag"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-blueprint-enterprise-rag-pipeline"
    notes: "Uses anyuid SCC RoleBinding for multiple infrastructure services (nv-ingest, minio, redis, zipkin) and NIMs"
    approach: "B"
---

# Security Context Constraints (SCC) for OpenShift

## Overview

OpenShift uses Security Context Constraints (SCCs) to control pod security policies. NVIDIA Blueprints often include components that require elevated privileges or specific UIDs that conflict with OpenShift's default `restricted-v2` SCC. This pattern shows how to create a custom SCC that allows necessary permissions while maintaining security.

## When to Use

- When blueprint components require `runAsUser: 0` (root)
- When containers specify non-numeric users in their Dockerfiles
- When you need to skip recursive SELinux relabeling on large PVCs (e.g., NIM model caches)
- When the default `restricted-v2` SCC blocks pod creation

## Common Error Messages

```
Error: container has runAsNonRoot and image has non-numeric user
```

```
0/N nodes are available: N node(s) had untolerated taint, N pod has unbound immediate PersistentVolumeClaims
```

## Custom SCC Pattern

### 1. SCC Resource Definition

Create a custom SCC that allows `runAsUser: 0` and skips SELinux relabeling:

**Example from openshift.yaml:**
```yaml
{{- if .Values.openshift.scc.create }}
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: {{ $fullName }}-nim
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service }}
  annotations:
    kubernetes.io/description: >-
      Custom SCC for VSS pods. Allows runAsUser 0 (needed by arango-db)
      and skips recursive SELinux relabeling on mounted volumes.
priority: {{ .Values.openshift.scc.priority | default 20 }}
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: null
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
readOnlyRootFilesystem: false
requiredDropCapabilities:
  - KILL
  - MKNOD
  - SETUID
  - SETGID
runAsUser:
  type: RunAsAny  # Allows containers to run as UID 0 or any other UID
seLinuxContext:
  type: RunAsAny  # Skips recursive SELinux relabeling on large PVCs
supplementalGroups:
  type: RunAsAny
users:
  - system:serviceaccount:{{ .Release.Namespace }}:default
  - system:serviceaccount:{{ .Release.Namespace }}:nim-cache-sa
  {{- range $nimKey, $nimVal := .Values.nimOperator }}
  {{- if and (kindIs "map" $nimVal) (hasKey $nimVal "service") }}
  - system:serviceaccount:{{ $.Release.Namespace }}:{{ $nimVal.service.name }}
  {{- end }}
  {{- end }}
volumes:
  - configMap
  - csi
  - downwardAPI
  - emptyDir
  - ephemeral
  - persistentVolumeClaim
  - projected
  - secret
{{- end }}
```

### 2. RoleBinding for SCC

Bind the SCC to service accounts in the namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ $fullName }}-nim-scc
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:{{ $fullName }}-nim
subjects:
  - kind: ServiceAccount
    name: default
    namespace: {{ .Release.Namespace }}
  - kind: ServiceAccount
    name: nim-cache-sa
    namespace: {{ .Release.Namespace }}
  {{- range $nimKey, $nimVal := .Values.nimOperator }}
  {{- if and (kindIs "map" $nimVal) (hasKey $nimVal "service") }}
  - kind: ServiceAccount
    name: {{ $nimVal.service.name }}
    namespace: {{ $.Release.Namespace }}
  {{- end }}
  {{- end }}
```

### 3. Values Configuration

Enable SCC creation in values:

**values-openshift.yaml:**
```yaml
openshift:
  enabled: true
  scc:
    create: true
    priority: 10  # Lower than restricted-v2 (priority 0), higher than anyuid
```

### 4. Conditional Rendering

Gate the SCC creation behind the `openshift.enabled` flag:

**templates/openshift.yaml:**
```yaml
{{- if .Values.openshift.enabled }}
{{- if .Values.openshift.scc.create }}
# ... SCC and RoleBinding resources
{{- end }}
{{- end }}
```

## Key SCC Fields Explained

| Field | Value | Reason |
|-------|-------|--------|
| `priority` | `10-20` | Higher than `restricted-v2` (0), lower than `anyuid` (10). SCC with highest priority wins. |
| `runAsUser.type` | `RunAsAny` | Allows containers to run as UID 0 or image-specified UID |
| `seLinuxContext.type` | `RunAsAny` | Skips recursive SELinux relabeling on large PVCs (critical for NIM caches) |
| `fsGroup.type` | `RunAsAny` | Allows any fsGroup setting |
| `allowPrivilegeEscalation` | `true` | Some containers need to escalate privileges |
| `allowPrivilegedContainer` | `false` | Don't allow fully privileged containers |
| `requiredDropCapabilities` | `[KILL, MKNOD, SETUID, SETGID]` | Drop dangerous capabilities |

## Service Account Strategy

### Explicitly List Service Accounts

The custom SCC explicitly grants permissions to specific service accounts:

1. **default**: For subchart components (Milvus, ArangoDB, Neo4j, etc.)
2. **nim-cache-sa**: For NIMCache pods (model downloading)
3. **NIMService SAs**: Dynamically generated from `nimOperator` values

### Dynamic NIMService SA Binding

The template automatically binds NIMService service accounts:

```yaml
{{- range $nimKey, $nimVal := .Values.nimOperator }}
{{- if and (kindIs "map" $nimVal) (hasKey $nimVal "service") }}
- system:serviceaccount:{{ $.Release.Namespace }}:{{ $nimVal.service.name }}
{{- end }}
{{- end }}
```

This ensures NIMs created via NIM Operator get the SCC automatically.

## Components Requiring Custom SCC

Based on multiple blueprint examples:

| Component | Reason | Default UID | SCC Field Needed | Blueprint Example |
|-----------|--------|-------------|------------------|-------------------|
| arango-db | Runs as root | 0 | `runAsUser: RunAsAny` | video-search-and-summarization |
| milvus | Runs as root or non-numeric user | 0 or `milvus` | `runAsUser: RunAsAny` | video-search-and-summarization |
| milvus-minio | Runs as root | 0 | `runAsUser: RunAsAny` | video-search-and-summarization |
| NIMCache (LLM) | Large PVC SELinux relabeling timeout | varies | `seLinuxContext: RunAsAny` | video-search-and-summarization |
| NIMCache (BioNeMo) | Very large PVC (2TB) with millions of files | varies | `seLinuxContext: RunAsAny` | generative-protein-binder-design |
| NIMService | May require specific UID | varies | `runAsUser: RunAsAny` | multiple |

**BioNeMo-specific considerations:**
- AlphaFold2 NIMCache: 2TB PVC, millions of genomic database files
- AlphaFold2-Multimer NIMCache: 2TB PVC, same database as AlphaFold2
- RFDiffusion NIMCache: 350GB PVC, model + TensorRT compilation cache
- ProteinMPNN NIMCache: 100GB PVC, standard model storage

## Priority Considerations

OpenShift SCCs have priorities that determine which SCC is selected:

| SCC Name | Priority | runAsUser | Use Case |
|----------|----------|-----------|----------|
| `privileged` | highest | Any | Cluster admins only |
| `anyuid` | 10 | Any | Legacy apps |
| **Custom SCC** | **10-20** | **Any** | **Blueprints with root containers** |
| `restricted-v2` | 0 | Random UID | Default for all pods |

**Recommendation:** Use priority `10` for custom SCCs to match `anyuid` but with more restrictive permissions.

---

## Approach B: anyuid SCC RoleBinding (from RAG Blueprint)

### When to Use This Approach

Use this simpler approach when:
- Components require `runAsUser: 0` but don't need special SELinux handling
- No large PVCs that would benefit from skipping SELinux relabeling
- You want a simpler deployment without creating custom SCC resources
- Infrastructure services (Redis, MinIO, PostgreSQL) need root access

**Use Approach A (Custom SCC) when:**
- Large PVCs (50GB+) need `seLinuxContext: RunAsAny` to skip relabeling
- You need fine-grained control over capabilities and volumes
- BioNeMo NIMs with multi-TB PVCs containing millions of files

### Pattern Overview

Instead of creating a custom SCC, bind service accounts to OpenShift's built-in `anyuid` SCC via a namespace-scoped RoleBinding.

**Benefits:**
- Simpler - single resource instead of SCC + RoleBinding
- No custom SCC management
- Works across OpenShift versions without SCC schema changes
- Namespace-scoped - no cluster-admin required to create SCC

**Trade-offs:**
- Doesn't skip SELinux relabeling (use Approach A if you need this)
- Less granular control over security permissions
- Uses built-in SCC that can't be customized

### 1. RoleBinding to anyuid SCC

**Example from RAG blueprint openshift.yaml:**
```yaml
{{- if .Values.openshift.enabled }}
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "nvidia-blueprint-rag.fullname" . }}-anyuid-scc
  labels:
    {{- include "nvidia-blueprint-rag.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:anyuid
subjects:
  - kind: ServiceAccount
    name: default
  - kind: ServiceAccount
    name: {{ include "nvidia-blueprint-rag.serviceAccountName" . }}
  - kind: ServiceAccount
    name: {{ .Release.Name }}-nv-ingest
  - kind: ServiceAccount
    name: {{ .Release.Name }}-minio
  - kind: ServiceAccount
    name: {{ .Release.Name }}-redis-master
  - kind: ServiceAccount
    name: {{ .Release.Name }}-redis-replica
  - kind: ServiceAccount
    name: nim-cache-sa
{{- if .Values.zipkin.enabled }}
  - kind: ServiceAccount
    name: {{ .Release.Name }}-zipkin
{{- end }}
{{- end }}
```

### 2. Dynamic Release Naming Pattern

**Critical fix from commit ca9844e:** Use `{{ .Release.Name }}-` prefix for subchart service accounts to support multiple blueprint instances in the same cluster.

**Before (hardcoded names):**
```yaml
subjects:
  - kind: ServiceAccount
    name: rag-nv-ingest  # ❌ Only works for release named "rag"
  - kind: ServiceAccount
    name: rag-minio
```

**After (dynamic names):**
```yaml
subjects:
  - kind: ServiceAccount
    name: {{ .Release.Name }}-nv-ingest  # ✅ Works for any release name
  - kind: ServiceAccount
    name: {{ .Release.Name }}-minio
```

**Exception:** `nim-cache-sa` is created by the NIM Operator with a fixed name (not by Helm), so it doesn't use the release prefix.

### 3. Values Configuration

**values-openshift.yaml:**
```yaml
openshift:
  enabled: true
  # No scc.create flag needed - just enable openshift support
```

### 4. Service Accounts Requiring anyuid SCC

Based on the RAG blueprint's infrastructure:

| Service Account | Component | Reason for anyuid |
|----------------|-----------|-------------------|
| `default` | Subchart defaults | Used by various subcharts |
| `{{ .Release.Name }}-nv-ingest` | NV-Ingest service | Document processing pipeline |
| `{{ .Release.Name }}-minio` | MinIO object storage | Runs as UID 0 |
| `{{ .Release.Name }}-redis-master` | Redis master | Bitnami Redis chart default UID |
| `{{ .Release.Name }}-redis-replica` | Redis replicas | Bitnami Redis chart default UID |
| `nim-cache-sa` | NIM Operator cache jobs | Created by NIM Operator |
| `{{ .Release.Name }}-zipkin` | Zipkin tracing | If observability enabled |

### 5. Conditional Observability Services

Gate optional service accounts behind feature flags:

```yaml
{{- if .Values.zipkin.enabled }}
  - kind: ServiceAccount
    name: {{ .Release.Name }}-zipkin
{{- end }}
```

This prevents binding SCC to service accounts that won't be created.

### Comparison: Approach A vs Approach B

| Aspect | Approach A (Custom SCC) | Approach B (anyuid RoleBinding) |
|--------|------------------------|--------------------------------|
| **Resources** | SCC + RoleBinding | RoleBinding only |
| **Complexity** | Higher | Lower |
| **SELinux Skip** | ✅ Yes (`seLinuxContext: RunAsAny`) | ❌ No |
| **Large PVCs** | ✅ Optimal for 50GB+ | ⚠️ May timeout on relabeling |
| **Customization** | ✅ Fine-grained control | ❌ Uses built-in anyuid |
| **Maintenance** | More complex | Simpler |
| **Use Case** | BioNeMo, large NIM caches | Standard RAG, infrastructure services |

### File Organization

```
deploy/helm/<chart-name>/
├── templates/
│   └── openshift.yaml         # Contains RoleBinding
└── values-openshift.yaml
    └── openshift:
          enabled: true
```

### Conversion from Approach A

If you have an existing blueprint using Approach A (custom SCC) and want to simplify to Approach B:

1. **Check PVC sizes:** If all PVCs < 50 GiB, consider Approach B
2. **Remove custom SCC definition** from openshift.yaml
3. **Replace with RoleBinding** to `system:openshift:scc:anyuid`
4. **Remove `scc.create` flag** from values
5. **List all service accounts** that need root access in RoleBinding subjects
6. **Test deployment** to ensure no SELinux relabeling timeouts

**When NOT to convert:** If you have NIM PVCs >50GB or BioNeMo databases, keep Approach A for `seLinuxContext: RunAsAny`.

---

## Known Issues and Gotchas

### Issue: SELinux Relabeling Timeout on Large PVCs

**Problem:** When a PVC is mounted, OpenShift recursively relabels all files with SELinux contexts. For NIM model caches (50-100 GiB), this can take hours and cause startup timeouts. For BioNeMo NIMs with 2TB PVCs containing millions of genomic database files (AlphaFold2), relabeling is essentially impossible.

**Error Messages:**
```
Warning: Failed to set pod label: timed out waiting for the condition
```

**Solution:** Set `seLinuxContext.type: RunAsAny` to skip relabeling:
```yaml
seLinuxContext:
  type: RunAsAny
```

**Real-world example from generative-protein-binder-design:**

AlphaFold2 requires a 2TB PVC with millions of genomic database files. The custom SCC explicitly documents the rationale:

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: {{ $fullName }}-nim
  annotations:
    kubernetes.io/description: >-
      Like nonroot, but skips recursive SELinux relabeling on volume mounts.
      Required for NIM services with very large cached model PVCs.
# ... rest of SCC definition ...
seLinuxContext:
  type: RunAsAny  # Prevents recursive relabeling
```

**From values-openshift.yaml comments:**
```yaml
# Creates a custom SCC (protein-design-nim) that is like "nonroot" but
# with seLinuxContext: RunAsAny. This prevents the kubelet from
# recursively relabeling every file on mounted volumes — critical for
# AlphaFold2's 2TB PVCs with millions of genomic database files.
scc:
  create: true
```

**When to use `seLinuxContext: RunAsAny`:**
- NIM model caches 50 GiB+ (LLM NIMs)
- NIM model caches 1-2 TB+ (BioNeMo NIMs)
- PVCs with millions of small files (genomic databases, model shards)
- Any PVC where recursive relabeling would take hours

### Issue: SCC Not Applied to Pods

**Problem:** Pods still fail with `runAsNonRoot` error even after SCC is created.

**Causes:**
1. Service account not listed in SCC's `users` field
2. RoleBinding not created or incorrect
3. Higher-priority SCC is being selected instead

**Debugging:**
```bash
# Check which SCC was applied to a pod
oc get pod <pod-name> -o yaml | grep "openshift.io/scc"

# Expected output:
openshift.io/scc: <release-name>-nim

# List all SCCs and their priorities
oc get scc -o custom-columns=NAME:.metadata.name,PRIORITY:.priority,RUNASUSER:.runAsUser.type
```

**Solution:** Verify service account is listed in SCC and RoleBinding exists:
```bash
oc get scc <release-name>-nim -o yaml
oc get rolebinding <release-name>-nim-scc -o yaml
```

### Issue: SCC Conflicts with Other Releases

**Problem:** Multiple Helm releases in the same namespace create conflicting SCCs.

**Solution:** Use release name in SCC name to avoid conflicts:
```yaml
name: {{ $fullName }}-nim  # $fullName = .Release.Name
```

### Issue: Pod Stuck in Pending Due to SCC

**Problem:** Pod remains in `Pending` state with event:
```
unable to validate against any security context constraint
```

**Cause:** Service account not bound to any SCC that allows the pod's security requirements.

**Solution:** Verify the RoleBinding exists and targets the correct service account:
```bash
oc describe pod <pod-name>
oc get rolebinding -n <namespace>
```

## Testing Notes

### Verify SCC Creation
```bash
oc get scc | grep <release-name>
oc describe scc <release-name>-nim
```

### Verify RoleBinding
```bash
oc get rolebinding -n <namespace> | grep scc
oc describe rolebinding <release-name>-nim-scc -n <namespace>
```

### Check Pod SCC Assignment
```bash
oc get pod <pod-name> -n <namespace> -o yaml | grep "openshift.io/scc"
```

Should output:
```
openshift.io/scc: <release-name>-nim
```

### Verify Service Account Permissions
```bash
oc adm policy who-can use scc/<release-name>-nim -n <namespace>
```

Should list the bound service accounts.

## Cleanup

SCCs are cluster-scoped resources but managed by Helm:

```bash
# Uninstall release (also deletes SCC)
helm uninstall <release-name> -n <namespace>

# Verify SCC deletion
oc get scc | grep <release-name>
```

## File Organization

```
deploy/helm/
└── <chart-name>/
    ├── templates/
    │   └── openshift.yaml         # SCC + RoleBinding
    └── values.yaml
        └── openshift:
              scc:
                create: true|false
                priority: <number>
```

## Best Practices

1. **Minimize Permissions**: Only grant `RunAsAny` for fields that actually need it
2. **Drop Capabilities**: Always drop unnecessary Linux capabilities
3. **No Privileged Containers**: Set `allowPrivilegedContainer: false` unless absolutely required
4. **Namespace-Scoped**: Use RoleBinding (not ClusterRoleBinding) to limit SCC to one namespace
5. **Explicit Service Accounts**: List specific service accounts rather than using `users: "*"`
6. **Document Justification**: Add annotations explaining why each permission is needed
7. **Priority Selection**: Use priority 10-20 (higher than restricted-v2, not cluster-admin level)
8. **Test Fallback**: Ensure blueprint works without custom SCC if possible (graceful degradation)

## Alternatives

### Alternative 1: Use Built-in `anyuid` SCC (See Approach B)

Binding service accounts to the built-in `anyuid` SCC is now documented as **Approach B** above. This is a valid conversion approach, not just an alternative.

**When to Use:** See "Approach B: anyuid SCC RoleBinding" section above.

### Alternative 2: Modify Container Images

**Pros:**
- Works with default `restricted-v2` SCC
- Better security posture

**Cons:**
- Not always feasible for third-party images
- Requires maintaining custom images
- Breaks official support

**When to Use:** Long-term production deployments where you control all images

## Conversion Checklist

When adding custom SCC to a blueprint:

- [ ] Create `templates/openshift.yaml` with SCC definition
- [ ] Set `runAsUser: RunAsAny` for containers requiring root
- [ ] Set `seLinuxContext: RunAsAny` for large PVCs
- [ ] Set appropriate priority (10-20)
- [ ] Create RoleBinding to bind SCC to service accounts
- [ ] List all required service accounts in SCC `users` field
- [ ] Add dynamic binding for NIMService SAs if using NIM Operator
- [ ] Gate SCC creation behind `openshift.scc.create` flag
- [ ] Document which components require the SCC and why
- [ ] Test pod creation and verify correct SCC is applied
- [ ] Verify SELinux relabeling doesn't timeout on large PVCs
