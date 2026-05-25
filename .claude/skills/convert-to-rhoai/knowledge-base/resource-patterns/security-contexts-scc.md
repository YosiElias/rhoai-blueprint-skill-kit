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

Based on video-search-and-summarization blueprint:

| Component | Reason | Default UID | SCC Field Needed |
|-----------|--------|-------------|------------------|
| arango-db | Runs as root | 0 | `runAsUser: RunAsAny` |
| milvus | Runs as root or non-numeric user | 0 or `milvus` | `runAsUser: RunAsAny` |
| milvus-minio | Runs as root | 0 | `runAsUser: RunAsAny` |
| NIMCache | Large PVC SELinux relabeling timeout | varies | `seLinuxContext: RunAsAny` |
| NIMService | May require specific UID | varies | `runAsUser: RunAsAny` |

## Priority Considerations

OpenShift SCCs have priorities that determine which SCC is selected:

| SCC Name | Priority | runAsUser | Use Case |
|----------|----------|-----------|----------|
| `privileged` | highest | Any | Cluster admins only |
| `anyuid` | 10 | Any | Legacy apps |
| **Custom SCC** | **10-20** | **Any** | **Blueprints with root containers** |
| `restricted-v2` | 0 | Random UID | Default for all pods |

**Recommendation:** Use priority `10` for custom SCCs to match `anyuid` but with more restrictive permissions.

## Known Issues and Gotchas

### Issue: SELinux Relabeling Timeout on Large PVCs

**Problem:** When a PVC is mounted, OpenShift recursively relabels all files with SELinux contexts. For NIM model caches (50-100 GiB), this can take hours and cause startup timeouts.

**Error Messages:**
```
Warning: Failed to set pod label: timed out waiting for the condition
```

**Solution:** Set `seLinuxContext.type: RunAsAny` to skip relabeling:
```yaml
seLinuxContext:
  type: RunAsAny
```

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

### Alternative 1: Use Built-in `anyuid` SCC

**Pros:**
- No custom SCC creation needed
- Widely available

**Cons:**
- Still requires RoleBinding to grant to service accounts
- Doesn't skip SELinux relabeling (can cause NIM PVC issues)
- May be overly permissive

**When to Use:** Simple blueprints without large PVCs or SELinux concerns

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
