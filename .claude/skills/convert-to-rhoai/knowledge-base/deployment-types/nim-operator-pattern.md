---
type: deployment-type
components: [nim-llm, nemo-embedding, nemo-rerank]
deployment_types: [helm]
resource_types: [gpu, storage]
architecture: []
source_examples:
  - blueprint: "video-search-and-summarization"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-video-search-and-summarization"
    notes: "Demonstrates NIM Operator integration for LLM, embedding, and reranking NIMs with OpenShift-specific configuration"
    approach: "A"
---

# NIM Operator Integration Pattern

## Overview

The NIM Operator integration pattern demonstrates how to deploy NVIDIA Inference Microservices (NIMs) on Red Hat OpenShift AI using the NIM Operator instead of traditional Kubernetes Deployments. This pattern is the **recommended approach** for deploying NIMs on OpenShift as it provides automated model lifecycle management, caching, and GPU resource allocation.

## When to Use

- Deploying NIMs (LLM, embedding, reranking, or other inference services) on OpenShift
- When the NIM Operator is installed and the `apps.nvidia.com/v1alpha1` API is available
- When you want automated model download, caching, and lifecycle management
- When you need dynamic PVC-based storage instead of hostPath volumes

## Key Benefits Over Deployment-Based Approach

1. **Automated Model Management**: NIMCache handles downloading and caching models from NGC
2. **PVC-Based Storage**: Uses dynamic PVCs instead of hostPath (required on OpenShift)
3. **Lifecycle Management**: NIMService manages replicas, GPU allocation, and health probes
4. **Cache Persistence**: NIMCache PVCs are annotated with `helm.sh/resource-policy: keep` to survive upgrades
5. **Integrated Secrets**: Automatically uses NGC pull secrets and API keys

## Conversion Pattern

### 1. Helm Conditional Gating

The NIM Operator templates are conditionally rendered based on:
1. NIM Operator API availability: `.Capabilities.APIVersions.Has "apps.nvidia.com/v1alpha1"`
2. NIM-specific enable flag: `$nim.enabled`

This allows the chart to work in both Operator and non-Operator environments.

**Example from nim-llm.yaml:**
```yaml
{{- $nim := index .Values.nimOperator "nim-llm" -}}
{{- if and (.Capabilities.APIVersions.Has "apps.nvidia.com/v1alpha1") (eq $nim.enabled true) }}
# ... NIMCache and NIMService resources
{{- end }}
```

### 2. NIMCache Resource

Each NIM gets a NIMCache that downloads and caches the model on a PVC.

**Structure:**
```yaml
apiVersion: apps.nvidia.com/v1alpha1
kind: NIMCache
metadata:
  name: {{ $nim.service.name }}-cache
  annotations:
    helm.sh/resource-policy: keep  # Survives helm uninstall and upgrades
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
      size: {{ $nim.storage.pvc.size | default "50Gi" }}
      volumeAccessMode: {{ $nim.storage.pvc.volumeAccessMode | default "ReadWriteOnce" }}
  {{- with $nim.tolerations }}
  tolerations:
{{ toYaml . | nindent 4 }}
  {{- end }}
```

**Key Fields:**
- `modelPuller`: Container image that pulls the model from NGC
- `pullSecret`: NGC Docker registry secret for image pulls
- `authSecret`: NGC API key secret for model downloads
- `storage.pvc`: PVC configuration (size, storageClass, access mode)
- `tolerations`: GPU node tolerations (passed through from NIM config)

### 3. NIMService Resource

NIMService runs the inference server and references the NIMCache.

**Structure:**
```yaml
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
      name: {{ $nim.service.name }}-cache  # References NIMCache
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
```

**Key Fields:**
- `storage.nimCache.name`: References the NIMCache resource
- `replicas`: Number of inference server replicas
- `resources`: GPU and other resource limits/requests
- `expose`: Service exposure configuration
- `env`: Environment variables (e.g., `TOKENIZERS_PARALLELISM: "false"`)
- `startupProbe`: Health probe configuration
- `tolerations`: GPU node tolerations
- `nodeSelector`: Node selection constraints

### 4. Values Schema

Each NIM has a consistent values schema under `nimOperator.<nim-name>`:

**Example (nim-llm):**
```yaml
nimOperator:
  nim-llm:
    enabled: true
    replicas: 1
    service:
      name: "llm-nim-svc"
    image:
      repository: nvcr.io/nim/meta/llama-3.1-8b-instruct
      tag: "latest"
      pullPolicy: Always
    resources:
      limits:
        nvidia.com/gpu: 1
      requests:
        nvidia.com/gpu: 1
    storage:
      pvc:
        create: true
        size: "100Gi"
        volumeAccessMode: ReadWriteOnce
        storageClass: ""  # Uses cluster default
    nodeSelector: {}
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
    env:
      - name: NIM_HTTP_API_PORT
        value: "8000"
      - name: TOKENIZERS_PARALLELISM
        value: "false"  # Prevents tokenizer race condition
    expose:
      service:
        name: http
        type: ClusterIP
        port: 8000
    startupProbe:
      enabled: true
      probe:
        httpGet:
          path: /v1/health/ready
          port: 8000
        initialDelaySeconds: 120
        periodSeconds: 30
        failureThreshold: 240  # Up to 2 hours for large model downloads
```

### 5. Disabling Subchart Deployments

When using NIM Operator, disable the traditional subchart Deployments:

```yaml
# Disable subchart Deployments
nim-llm:
  enabled: false

nemo-embedding:
  enabled: false

nemo-rerank:
  enabled: false
```

### 6. Required Secrets

The NIM Operator requires two types of secrets:

**Image Pull Secret:**
```yaml
nvcf:
  dockerRegSecrets:
    - name: ngc-docker-reg-secret
      username: "$oauthtoken"
      password: ""  # Set via --set at install time
```

**NGC API Key Secret:**
```yaml
nvcf:
  additionalSecrets:
    - name: ngc-api-key-secret
      stringData:
        key: NGC_API_KEY
        value: ""  # Set via --set at install time
```

**Global References:**
```yaml
global:
  ngcImagePullSecretName: ngc-docker-reg-secret

imagePullSecret:
  name: ngc-docker-reg-secret

ngcApiSecret:
  name: ngc-api-key-secret
```

### 7. Service Account Linking

Link the NGC pull secret to the NIM Operator service account:

```bash
oc create sa nim-cache-sa -n $NAMESPACE || true
oc secrets link nim-cache-sa ngc-docker-reg-secret --for=pull -n $NAMESPACE
```

## PVC Sizing Guidance

Based on the video-search-and-summarization blueprint:

| NIM Type | Model | PVC Size | Notes |
|----------|-------|----------|-------|
| nim-llm | llama-3.1-8b-instruct | 100 GiB | LLM models require substantial space |
| nemo-embedding | llama-3.2-nv-embedqa-1b-v2 | 50 GiB | Embedding models are smaller |
| nemo-rerank | llama-3.2-nv-rerankqa-1b-v2 | 50 GiB | Reranking models are smaller |

**Important:** PVCs created by NIMCache are immutable. To resize, delete the NIMCache and PVC, then re-run `helm install`.

## Startup Probe Configuration

NIMs have long startup times due to model downloads. Configure generous startup probes:

| NIM Type | Initial Delay | Period | Failure Threshold | Max Wait Time |
|----------|---------------|--------|-------------------|---------------|
| nim-llm | 120s | 30s | 240 | ~2 hours |
| nemo-embedding | 60s | 30s | 120 | ~1 hour |
| nemo-rerank | 60s | 30s | 120 | ~1 hour |

## Known Issues and Gotchas

### Issue: TOKENIZERS_PARALLELISM Race Condition

**Problem:** HuggingFace tokenizers library has a thread pool race condition that can cause NIMs to crash or fail startup probes intermittently.

**Solution:** Set `TOKENIZERS_PARALLELISM=false` for all NIMs:
```yaml
env:
  - name: TOKENIZERS_PARALLELISM
    value: "false"
```

### Issue: NIMCache PVC Undersizing

**Problem:** NIM model cache PVCs must be large enough to hold all downloaded model profiles. Undersized PVCs cause download failures with unclear error messages.

**Solution:** Use recommended PVC sizes (100 GiB for LLMs, 50 GiB for embedding/reranking). Monitor NIMCache status:
```bash
oc get nimcache -n $NAMESPACE -w
```

### Issue: Model Download Monitoring

**Problem:** Model downloads can take 10-60 minutes. Users may think the deployment is stuck.

**Solution:** Monitor NIMCache status until all caches show `Ready`:
```bash
oc get nimcache -n $NAMESPACE -w
```

Expected output:
```
NAME                                                          STATUS   AGE
llm-nim-svc-cache                                             Ready    30m
nemo-embedding-embedding-deployment-embedding-service-cache   Ready    15m
nemo-rerank-ranking-deployment-ranking-service-cache          Ready    15m
```

### Issue: NIMCache Persistence After Uninstall

**Problem:** NIMCache PVCs persist after `helm uninstall` due to `helm.sh/resource-policy: keep` annotation.

**Solution:** This is intentional to avoid re-downloading large models. To clean up:
```bash
oc delete nimcache --all -n $NAMESPACE
oc delete pvc -l app.nvidia.com/nim-cache -n $NAMESPACE
```

## Dependencies

- NIM Operator installed (`apps.nvidia.com/v1alpha1` API available)
- NGC Docker registry secret created and linked to service accounts
- NGC API key secret created
- GPU nodes with appropriate tolerations
- Dynamic storage provisioner (PVCs)

## Testing Notes

### Verify NIM Operator Installation
```bash
oc get crd | grep nim
# Should show nimcaches.apps.nvidia.com and nimservices.apps.nvidia.com
```

### Verify NIMCache Status
```bash
oc get nimcache -n $NAMESPACE
oc describe nimcache <name> -n $NAMESPACE
```

### Verify NIMService Status
```bash
oc get nimservice -n $NAMESPACE
oc describe nimservice <name> -n $NAMESPACE
```

### Test Inference Endpoint
```bash
oc exec -n $NAMESPACE deployment/llm-nim-svc -- \
  curl -s http://localhost:8000/v1/health/ready
```

### Check Model Download Progress
```bash
oc logs -n $NAMESPACE -l app=nimcache -f
```

## File Organization

For a blueprint using this pattern:

```
deploy/helm/
├── nvidia-blueprint-<name>-<version>.tgz
│   └── nvidia-blueprint-<name>/
│       ├── templates/
│       │   ├── nim-llm.yaml           # NIMCache + NIMService for LLM
│       │   ├── nim-embedding.yaml     # NIMCache + NIMService for embedding
│       │   ├── nim-reranking.yaml     # NIMCache + NIMService for reranking
│       │   └── nvcf_secrets.yaml      # NGC secrets template
│       └── values.yaml                # Base values
└── values-openshift.yaml              # OpenShift overlay with nimOperator config
```

## Conversion Checklist

When converting a blueprint to use NIM Operator:

- [ ] Create NIM template files (nim-llm.yaml, nim-embedding.yaml, etc.)
- [ ] Add nimOperator block to values.yaml for each NIM
- [ ] Disable subchart NIM Deployments (set `nim-llm.enabled: false`, etc.)
- [ ] Configure NGC secrets (imagePullSecret, ngcApiSecret)
- [ ] Set appropriate PVC sizes for model caching
- [ ] Configure GPU tolerations for each NIM
- [ ] Set `TOKENIZERS_PARALLELISM=false` in env vars
- [ ] Configure generous startup probes (account for model download time)
- [ ] Document service account linking steps in deployment guide
- [ ] Test model download and caching workflow
- [ ] Verify health endpoints are accessible
