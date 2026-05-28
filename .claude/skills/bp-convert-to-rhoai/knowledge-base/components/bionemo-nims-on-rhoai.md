---
type: component
components: [bionemo-nims, alphafold2, rfdiffusion, proteinmpnn]
deployment_types: [helm]
resource_types: [gpu, storage]
architecture: []
source_examples:
  - blueprint: "generative-protein-binder-design"
    source_repo: "https://github.com/NVIDIA-BioNeMo-blueprints/generative-protein-binder-design"
    fork_repo: "https://github.com/rh-ai-quickstart/generative-protein-binder-design"
    notes: "Complete BioNeMo NIM deployment with AlphaFold2, RFDiffusion, ProteinMPNN, and AlphaFold2-Multimer"
---

# BioNeMo NIMs on RHOAI

## Overview

BioNeMo NIMs are NVIDIA's Biology-specific Inference Microservices for computational biology and drug discovery workflows. Unlike LLM NIMs, BioNeMo NIMs have unique requirements:

- Very large PVC requirements (100GB - 2TB per NIM)
- Extended startup probe times (up to 6 hours for TensorRT compilation)
- Large genomic database files (millions of small files for AlphaFold2)
- Specialized inference workloads (protein structure prediction, molecular generation)

## When to Use

- Deploying NVIDIA BioNeMo blueprints on OpenShift/RHOAI
- Protein structure prediction workflows (AlphaFold2, AlphaFold2-Multimer)
- Protein design workflows (RFDiffusion, ProteinMPNN)
- Molecular docking and virtual screening (DiffDock)
- Molecular generation (MolMIM, GenMol)

## BioNeMo NIM Catalog

### Protein Structure Prediction

#### AlphaFold2

Predicts protein structure from amino acid sequence using deep learning.

**Image:** `nvcr.io/nim/deepmind/alphafold2:2.1`

**Resource Requirements:**
```yaml
resources:
  limits:
    nvidia.com/gpu: 1
  requests:
    nvidia.com/gpu: 1
storage:
  pvc:
    size: "2000Gi"  # 2TB for genomic database
    volumeAccessMode: ReadWriteOnce
```

**Startup Probe:**
```yaml
startupProbe:
  enabled: true
  probe:
    httpGet:
      path: /v1/health/ready
      port: 8000
    initialDelaySeconds: 120
    periodSeconds: 30
    failureThreshold: 720  # Up to 6 hours for database download
```

**Environment Variables:**
```yaml
env:
  - name: NIM_HTTP_API_PORT
    value: "8000"
  - name: NIM_CACHE_PATH
    value: "/model-store"
  - name: TOKENIZERS_PARALLELISM
    value: "false"
```

**Critical Notes:**
- 2TB PVC contains millions of genomic database files
- Requires custom SCC with `seLinuxContext: RunAsAny` to prevent recursive relabeling timeout
- Database download can take hours on first deployment
- Use `helm.sh/resource-policy: keep` on NIMCache to preserve database across upgrades

#### AlphaFold2-Multimer

Predicts structure of protein complexes (multiple chains).

**Image:** `nvcr.io/nim/deepmind/alphafold2-multimer:2.1`

**Resource Requirements:**
```yaml
resources:
  limits:
    nvidia.com/gpu: 1
  requests:
    nvidia.com/gpu: 1
storage:
  pvc:
    size: "2000Gi"  # Same database as AlphaFold2
    volumeAccessMode: ReadWriteOnce
```

**Startup Probe:**
```yaml
startupProbe:
  enabled: true
  probe:
    httpGet:
      path: /v1/health/ready
      port: 8000
    initialDelaySeconds: 120
    periodSeconds: 30
    failureThreshold: 360  # Up to 3 hours
```

**Notes:**
- Uses the same genomic database as AlphaFold2 (2TB)
- Same SELinux relabeling concerns as AlphaFold2
- Can share database PVC with AlphaFold2 in some deployments (not recommended with NIM Operator)

### Protein Design

#### RFDiffusion

Generates protein backbones using generative diffusion models.

**Image:** `nvcr.io/nim/ipd/rfdiffusion:2.0`

**Resource Requirements:**
```yaml
resources:
  limits:
    nvidia.com/gpu: 1
  requests:
    nvidia.com/gpu: 1
storage:
  pvc:
    size: "350Gi"
    volumeAccessMode: ReadWriteOnce
```

**Startup Probe:**
```yaml
startupProbe:
  enabled: true
  probe:
    httpGet:
      path: /v1/health/ready
      port: 8000
    initialDelaySeconds: 120
    periodSeconds: 30
    failureThreshold: 720  # Up to 6 hours for TensorRT compilation
```

**Environment Variables:**
```yaml
env:
  - name: NIM_HTTP_API_PORT
    value: "8000"
  - name: NIM_CACHE_PATH
    value: "/home/nvs/.cache/nim/models"
  - name: TOKENIZERS_PARALLELISM
    value: "false"
```

**Critical Gotcha - TensorRT Compilation:**

RFDiffusion performs a one-time TensorRT engine compilation on first startup for the specific GPU architecture:

- **First run:** 2-6 hours compilation time
- **Subsequent runs:** Fast startup using cached engines
- **Pod rollout:** Forces recompilation (engines are in PVC but may be architecture-specific)
- **Health endpoint:** Blocks `/v1/health/ready` until compilation completes

**From values-openshift.yaml:**
```yaml
# RFDiffusion compiles TensorRT engines on FIRST-EVER start for a given
# GPU architecture. This one-time build can take 2-4 hours. The compiled
# engines are cached in ephemeral container storage, so subsequent pod
# restarts on the same node reuse them — but a rollout (new ReplicaSet)
# loses the cache and forces a full rebuild.
# failureThreshold: 720 × periodSeconds: 30 = 6 hours headroom.
```

**Why such a high failureThreshold:**
- TensorRT compilation is GPU-architecture specific
- Compilation happens at startup, before health check passes
- Without high threshold, pod gets killed before compilation completes
- 6-hour window accommodates worst-case compilation time

#### ProteinMPNN

Designs amino acid sequences for given protein backbones.

**Image:** `nvcr.io/nim/ipd/proteinmpnn:1.0`

**Resource Requirements:**
```yaml
resources:
  limits:
    nvidia.com/gpu: 1
  requests:
    nvidia.com/gpu: 1
storage:
  pvc:
    size: "100Gi"  # Smallest BioNeMo NIM
    volumeAccessMode: ReadWriteOnce
```

**Startup Probe:**
```yaml
startupProbe:
  enabled: true
  probe:
    httpGet:
      path: /v1/health/ready
      port: 8000
    initialDelaySeconds: 120
    periodSeconds: 30
    failureThreshold: 120  # ~1 hour (standard model download)
```

**Environment Variables:**
```yaml
env:
  - name: NIM_HTTP_API_PORT
    value: "8000"
  - name: NIM_CACHE_PATH
    value: "/home/nvs/.cache/nim/models"
  - name: TOKENIZERS_PARALLELISM
    value: "false"
```

**Notes:**
- Smallest BioNeMo NIM by storage requirements (100GB)
- Standard startup time similar to LLM NIMs
- No TensorRT compilation delay like RFDiffusion

## Multi-NIM Workflow Pattern

BioNeMo blueprints often deploy multiple NIMs in a pipeline. Example from generative-protein-binder-design:

**Workflow:** Target sequence → AlphaFold2 (structure) → RFDiffusion (binder backbone) → ProteinMPNN (binder sequence) → AlphaFold2-Multimer (complex structure)

**Total Resource Requirements:**
```yaml
# 4 NIMs × 1 GPU each = 4 GPUs minimum
# 2TB + 350GB + 100GB + 2TB = ~4.5TB storage
# 6 hours initial startup (longest NIM determines readiness)
```

**values-openshift.yaml structure:**
```yaml
nimOperator:
  alphafold2:
    enabled: true
    replicas: 1
    service:
      name: "alphafold2"
    storage:
      pvc:
        size: "2000Gi"
    resources:
      limits:
        nvidia.com/gpu: 1
    startupProbe:
      enabled: true
      probe:
        failureThreshold: 720
  
  rfdiffusion:
    enabled: true
    replicas: 1
    service:
      name: "rfdiffusion"
    storage:
      pvc:
        size: "350Gi"
    resources:
      limits:
        nvidia.com/gpu: 1
    startupProbe:
      enabled: true
      probe:
        failureThreshold: 720
  
  proteinmpnn:
    enabled: true
    replicas: 1
    service:
      name: "proteinmpnn"
    storage:
      pvc:
        size: "100Gi"
    resources:
      limits:
        nvidia.com/gpu: 1
    startupProbe:
      enabled: true
      probe:
        failureThreshold: 120
  
  alphafold2-multimer:
    enabled: true
    replicas: 1
    service:
      name: "alphafold2-multimer"
    storage:
      pvc:
        size: "2000Gi"
    resources:
      limits:
        nvidia.com/gpu: 1
    startupProbe:
      enabled: true
      probe:
        failureThreshold: 360
```

## GPU Tolerations

BioNeMo NIMs require GPU nodes. Add tolerations for GPU node taints:

```yaml
tolerations:
  - key: g6-gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  - key: p4-gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  # Or use generic GPU toleration:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

**Apply to both NIMCache and NIMService** to ensure model download pods and inference pods can schedule on GPU nodes.

## Custom SCC Requirements

BioNeMo NIMs with large PVCs require a custom SCC:

**templates/openshift.yaml:**
```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: {{ $fullName }}-nim
  annotations:
    kubernetes.io/description: >-
      Like nonroot, but skips recursive SELinux relabeling on volume mounts.
      Required for NIM services with very large cached model PVCs.
priority: 20
# ... standard SCC fields ...
seLinuxContext:
  type: RunAsAny  # CRITICAL: Prevents recursive relabeling
users:
  - system:serviceaccount:{{ .Release.Namespace }}:default
  - system:serviceaccount:{{ .Release.Namespace }}:nim-cache-sa
  {{- range $nimKey, $nimVal := .Values.nimOperator }}
  {{- if and (kindIs "map" $nimVal) (hasKey $nimVal "service") }}
  - system:serviceaccount:{{ $.Release.Namespace }}:{{ $nimVal.service.name }}
  {{- end }}
  {{- end }}
```

**Why seLinuxContext: RunAsAny is required:**
- AlphaFold2: 2TB PVC with millions of genomic database files
- Recursive SELinux relabeling would take hours/days
- Without RunAsAny, kubelet attempts relabeling and times out
- Pod never starts, stuck in "ContainerCreating" state

## NIM Template Structure

Each BioNeMo NIM gets a dedicated template file:

**templates/nim-alphafold2.yaml:**
```yaml
{{- $nim := index .Values.nimOperator "alphafold2" -}}
{{- if and (.Capabilities.APIVersions.Has "apps.nvidia.com/v1alpha1") (eq $nim.enabled true) }}
apiVersion: apps.nvidia.com/v1alpha1
kind: NIMCache
metadata:
  name: {{ $nim.service.name }}-cache
  annotations:
    helm.sh/resource-policy: keep
spec:
  source:
    ngc:
      modelPuller: "{{ $nim.image.repository }}:{{ $nim.image.tag }}"
      pullSecret: {{ .Values.imagePullSecret.name }}
      authSecret: {{ .Values.ngcApiSecret.name }}
  storage:
    pvc:
      create: {{ $nim.storage.pvc.create | default true }}
      {{- if $nim.storage.pvc.storageClass }}
      storageClass: {{ $nim.storage.pvc.storageClass }}
      {{- end }}
      size: {{ $nim.storage.pvc.size | default "2000Gi" }}
      volumeAccessMode: {{ $nim.storage.pvc.volumeAccessMode | default "ReadWriteOnce" }}
  {{- with $nim.tolerations }}
  tolerations:
{{ toYaml . | nindent 4 }}
  {{- end }}
---
apiVersion: apps.nvidia.com/v1alpha1
kind: NIMService
metadata:
  name: {{ $nim.service.name }}
spec:
  image:
    repository: {{ $nim.image.repository }}
    tag: "{{ $nim.image.tag }}"
    pullPolicy: {{ $nim.image.pullPolicy | default "Always" }}
    pullSecrets:
      - {{ .Values.imagePullSecret.name }}
  authSecret: {{ .Values.ngcApiSecret.name }}
  storage:
    nimCache:
      name: {{ $nim.service.name }}-cache
  replicas: {{ $nim.replicas | default 1 }}
  resources:
{{ toYaml $nim.resources | nindent 4 }}
  expose:
{{ toYaml $nim.expose | nindent 4 }}
  env:
{{ toYaml $nim.env | nindent 4 }}
  {{- with $nim.startupProbe }}
  {{- if .enabled }}
  startupProbe:
    enabled: true
    probe:
{{ toYaml .probe | nindent 6 }}
  {{- end }}
  {{- end }}
  {{- with $nim.tolerations }}
  tolerations:
{{ toYaml . | nindent 4 }}
  {{- end }}
  {{- with $nim.nodeSelector }}
  nodeSelector:
{{ toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

**Pattern for each NIM:**
- Conditional rendering based on NIM Operator API + enabled flag
- NIMCache with `helm.sh/resource-policy: keep` annotation
- NIMService references NIMCache by name
- GPU tolerations passed to both NIMCache and NIMService
- Startup probe configuration based on NIM-specific requirements

## OpenShift Routes

Expose BioNeMo NIMs via OpenShift Routes:

**templates/openshift.yaml:**
```yaml
{{- if .Values.openshift.enabled }}
{{- $routes := .Values.openshift.routes | default dict }}
{{- $services := list "alphafold2" "rfdiffusion" "proteinmpnn" "alphafold2multimer" }}

{{- range $serviceName := $services }}
{{- $routeConfig := index $routes $serviceName }}
{{- if (default false $routeConfig.enabled) }}
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ $fullName }}-{{ $serviceName }}
spec:
  to:
    kind: Service
    name: {{ $fullName }}-{{ $serviceName }}
  port:
    targetPort: {{ $serviceName }}-port
  {{- with $routeConfig.tls }}
  tls:
    termination: {{ .termination | default "edge" }}
    {{- if .insecureEdgeTerminationPolicy }}
    insecureEdgeTerminationPolicy: {{ .insecureEdgeTerminationPolicy }}
    {{- end }}
  {{- end }}
{{- end }}
{{- end }}
{{- end }}
```

**values-openshift.yaml:**
```yaml
openshift:
  enabled: true
  routes:
    alphafold2:
      enabled: true
      tls:
        termination: edge
        insecureEdgeTerminationPolicy: Redirect
    rfdiffusion:
      enabled: true
      tls:
        termination: edge
        insecureEdgeTerminationPolicy: Redirect
    # ... etc for other NIMs
```

## Deployment Workflow

### 1. Install NIM Operator

Verify NIM Operator is installed:
```bash
oc get crd | grep nim
# Should show nimcaches.apps.nvidia.com and nimservices.apps.nvidia.com
```

### 2. Create NGC Secrets

```bash
export NGC_API_KEY="<your-ngc-api-key>"

# NGC Docker registry secret
oc create secret docker-registry ngc-secret \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password="${NGC_API_KEY}" \
  -n $NAMESPACE

# NGC API secret
oc create secret generic ngc-api \
  --from-literal=NGC_API_KEY="${NGC_API_KEY}" \
  -n $NAMESPACE
```

### 3. Install Helm Chart

```bash
helm install protein-design . \
  -f values.yaml \
  -f values-openshift.yaml \
  -n $NAMESPACE
```

### 4. Monitor NIMCache Downloads

```bash
oc get nimcache -n $NAMESPACE -w
```

Wait for all caches to show `Ready`:
```
NAME                       STATUS   AGE
alphafold2-cache           Ready    45m
rfdiffusion-cache          Ready    38m
proteinmpnn-cache          Ready    15m
alphafold2-multimer-cache  Ready    48m
```

### 5. Verify NIMServices

```bash
oc get nimservice -n $NAMESPACE
oc get pods -n $NAMESPACE
```

### 6. Test Health Endpoints

```bash
for svc in alphafold2 rfdiffusion proteinmpnn alphafold2-multimer; do
  echo "--- ${svc} ---"
  oc exec deploy/${svc} -n $NAMESPACE -- \
    curl -s http://localhost:8000/v1/health/ready
  echo
done
```

## Notebook Integration

BioNeMo workflows typically run in Jupyter notebooks. Use DEPLOYMENT_MODE pattern:

**notebook cell:**
```python
import os

DEPLOYMENT_MODE = os.getenv('DEPLOYMENT_MODE', 'local')

if DEPLOYMENT_MODE == 'openshift':
    # Cluster-internal K8s DNS names (matches NIM Operator service names)
    NIM_HOSTS = {
        'ALPHAFOLD2': os.getenv('ALPHAFOLD2_HOST', 'http://alphafold2'),
        'RFDIFFUSION': os.getenv('RFDIFFUSION_HOST', 'http://rfdiffusion'),
        'PROTEINMPNN': os.getenv('PROTEINMPNN_HOST', 'http://proteinmpnn'),
        'AF2_MULTIMER': os.getenv('AF2_MULTIMER_HOST', 'http://alphafold2-multimer'),
    }
    # All services use port 8000 on OpenShift
    PORT = 8000
else:
    # Localhost with different ports per service
    NIM_HOSTS = {
        'ALPHAFOLD2': 'http://localhost',
        'RFDIFFUSION': 'http://localhost',
        'PROTEINMPNN': 'http://localhost',
        'AF2_MULTIMER': 'http://localhost',
    }
    PORTS = {'ALPHAFOLD2': 8081, 'RFDIFFUSION': 8082, 'PROTEINMPNN': 8083, 'AF2_MULTIMER': 8084}
```

**Create RHOAI workbench with environment variable:**
- Set `DEPLOYMENT_MODE=openshift` when creating workbench
- Notebook automatically uses cluster-internal DNS names
- No port-forwarding or Routes needed for notebook access

## Known Issues and Gotchas

### Issue: AlphaFold2 Pod Stuck in ContainerCreating

**Symptom:** Pod never starts, stuck in "ContainerCreating" for hours.

**Cause:** SELinux recursive relabeling on 2TB PVC with millions of files.

**Solution:** Custom SCC with `seLinuxContext: RunAsAny` (see Custom SCC Requirements above).

### Issue: RFDiffusion Pod Killed During Startup

**Symptom:** RFDiffusion pod restarts repeatedly, never becomes ready.

**Cause:** TensorRT compilation takes 2-6 hours, startup probe fails before compilation completes.

**Solution:** Set `startupProbe.failureThreshold: 720` (6 hours).

### Issue: NIMCache Download Appears Stuck

**Symptom:** `oc get nimcache` shows cache in "Pending" or "Downloading" for hours.

**Cause:** Large model downloads (2TB for AlphaFold2) take time.

**Solution:** Monitor logs, be patient:
```bash
oc logs -n $NAMESPACE -l app=nimcache -f
```

### Issue: Multiple NIMs Not All Ready

**Symptom:** Some NIMs are ready, others still starting.

**Cause:** Different startup times (AlphaFold2: 6h, ProteinMPNN: 1h).

**Solution:** Wait for all NIMs to be ready before running workflow. Check comprehensive health in notebook:
```python
import requests, time

def check_all_nims_health(retries=3, delay=10):
    nims = [
        ("AlphaFold2", "http://alphafold2:8000/v1/health/ready"),
        ("RFDiffusion", "http://rfdiffusion:8000/v1/health/ready"),
        ("ProteinMPNN", "http://proteinmpnn:8000/v1/health/ready"),
        ("AlphaFold2-Multimer", "http://alphafold2-multimer:8000/v1/health/ready"),
    ]
    all_ready = True
    for name, url in nims:
        for attempt in range(retries):
            try:
                resp = requests.get(url, timeout=10)
                if resp.json().get("status") == "ready":
                    print(f"  {name}: Ready")
                    break
            except Exception:
                if attempt < retries - 1:
                    time.sleep(delay)
        else:
            all_ready = False
            print(f"  {name}: NOT READY")
    return all_ready
```

## Testing Checklist

- [ ] NIM Operator installed and CRDs available
- [ ] NGC secrets created with Helm ownership labels
- [ ] Helm chart installs without errors
- [ ] All NIMCache resources reach "Ready" status
- [ ] All NIMService resources reach "Ready" status
- [ ] All pods running without restarts
- [ ] Custom SCC created and bound to service accounts
- [ ] Health endpoints respond with "ready" status
- [ ] OpenShift Routes accessible (if enabled)
- [ ] Notebook can connect to all NIMs (if using notebook pattern)
- [ ] Workflow completes end-to-end (all NIMs in pipeline)

## Related Patterns

- [deployment-types/nim-operator-pattern.md](../deployment-types/nim-operator-pattern.md) - General NIM Operator integration
- [resource-patterns/security-contexts-scc.md](../resource-patterns/security-contexts-scc.md) - Custom SCC for large PVCs
- [deployment-types/rhoai-notebook-pattern.md](../deployment-types/rhoai-notebook-pattern.md) - Notebook integration with DEPLOYMENT_MODE
