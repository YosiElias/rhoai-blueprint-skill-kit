---
type: resource-pattern
components: []
deployment_types: [helm]
resource_types: [gpu]
architecture: []
source_examples:
  - blueprint: "video-search-and-summarization"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-video-search-and-summarization"
    notes: "Demonstrates GPU resource allocation and tolerations for NIM services and VSS application"
    approach: "A"
---

# GPU Resource Allocation on OpenShift

## Overview

OpenShift GPU nodes typically have taints that prevent non-GPU workloads from scheduling on expensive GPU hardware. This pattern shows how to configure GPU resource requests/limits and tolerations to schedule GPU workloads correctly.

## When to Use

- Workloads requiring NVIDIA GPUs (inference, training, video processing)
- Scheduling on nodes with `nvidia.com/gpu` taints
- Multi-GPU or fractional GPU allocation
- Preventing GPU workloads from running on CPU-only nodes

## Prerequisites

- NVIDIA GPU Operator installed on OpenShift cluster
- GPU nodes labeled with `nvidia.com/gpu.present=true`
- Nodes may have taints like `nvidia.com/gpu:NoSchedule`

**Verify GPU availability:**
```bash
oc get nodes -l nvidia.com/gpu.present=true
oc describe node <gpu-node> | grep -A 5 "Allocatable"
```

Expected output:
```
Allocatable:
  nvidia.com/gpu: 1  (or higher)
```

## GPU Resource Allocation Pattern

### 1. GPU Resource Requests and Limits

Specify GPU count in pod resources:

**For NIM services (via NIM Operator):**
```yaml
nimOperator:
  nim-llm:
    resources:
      limits:
        nvidia.com/gpu: 1
      requests:
        nvidia.com/gpu: 1
```

**For standard Deployments/Pods:**
```yaml
resources:
  limits:
    nvidia.com/gpu: 1
  requests:
    nvidia.com/gpu: 1
```

**Key Points:**
- Use `nvidia.com/gpu` resource name (not `nvidia.com/gpu.count` or variations)
- Set both `requests` and `limits` to same value for guaranteed allocation
- GPUs are whole-unit resources (no fractional allocation unless using MIG)

### 2. GPU Tolerations

Add tolerations to schedule on nodes with GPU taints:

**For NIM services:**
```yaml
nimOperator:
  nim-llm:
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

**For standard workloads:**
```yaml
spec:
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
```

**Common GPU Taints:**

| Taint Key | Effect | Operator | Meaning |
|-----------|--------|----------|---------|
| `nvidia.com/gpu` | NoSchedule | Exists | Node has GPUs, only GPU pods allowed |
| `nvidia.com/gpu` | NoExecute | Exists | Evict non-GPU pods from GPU nodes |
| Custom (e.g., `gpu-workload`) | NoSchedule | Equal:value | Cluster-specific GPU taint |

### 3. Node Selector (Optional)

Force scheduling on GPU nodes:

```yaml
nodeSelector:
  nvidia.com/gpu.present: "true"
```

**When to Use:**
- When tolerations alone don't guarantee GPU node scheduling
- When you need specific GPU types (L40S vs A100 vs H100)

**Example with GPU type:**
```yaml
nodeSelector:
  nvidia.com/gpu.product: NVIDIA-L40S
```

### 4. Example: VSS Application GPU Configuration

**values-openshift.yaml:**
```yaml
vss:
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
    # Add cluster-specific tolerations as needed:
    # - key: <your-gpu-taint-key>
    #   operator: Equal
    #   value: "true"
    #   effect: NoSchedule
  resources:
    limits:
      nvidia.com/gpu: 1
```

### 5. Example: NIM GPU Configuration

**values-openshift.yaml:**
```yaml
nimOperator:
  nim-llm:
    enabled: true
    replicas: 1
    resources:
      limits:
        nvidia.com/gpu: 1
      requests:
        nvidia.com/gpu: 1
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
    # Optional: specific GPU type
    nodeSelector: {}
    
  nemo-embedding:
    enabled: true
    resources:
      limits:
        nvidia.com/gpu: 1
      requests:
        nvidia.com/gpu: 1
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

## Multi-GPU Allocation

### Multiple GPUs for One Pod

For models requiring tensor parallelism:

```yaml
resources:
  limits:
    nvidia.com/gpu: 4
  requests:
    nvidia.com/gpu: 4
```

**Example: LLM with 4-GPU tensor parallelism:**
```yaml
nimOperator:
  nim-llm:
    resources:
      limits:
        nvidia.com/gpu: 4
      requests:
        nvidia.com/gpu: 4
```

**Scheduling Constraint:** All GPUs must be on the same node unless using multi-node training frameworks.

### Multiple Single-GPU Pods

For horizontal scaling (e.g., multiple inference replicas):

```yaml
nimOperator:
  nim-llm:
    replicas: 4  # 4 pods, each with 1 GPU
    resources:
      limits:
        nvidia.com/gpu: 1
```

**Total GPUs:** `replicas × nvidia.com/gpu` (4 GPUs in example)

**Scheduling:** Pods can spread across multiple GPU nodes

## Checking GPU Node Taints

Identify GPU taints in your cluster:

```bash
# List all GPU nodes and their taints
oc get nodes -l nvidia.com/gpu.present=true -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

Example output:
```
NAME                                       TAINTS
ip-10-0-1-100.ec2.internal                [map[effect:NoSchedule key:nvidia.com/gpu value:present]]
ip-10-0-1-101.ec2.internal                [map[effect:NoSchedule key:nvidia.com/gpu value:present]]
```

**Extracting taint details:**
```bash
oc describe node <gpu-node-name> | grep Taints
```

Example output:
```
Taints:             nvidia.com/gpu=present:NoSchedule
```

**Corresponding toleration:**
```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: present
    effect: NoSchedule
```

**Or use wildcard:**
```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

## GPU Models and Requirements

Based on video-search-and-summarization blueprint:

| Component | Model | GPU Count | VRAM Required | Reason |
|-----------|-------|-----------|---------------|--------|
| nim-llm | llama-3.1-8b-instruct | 1 | 16 GB | LLM inference |
| nim-llm (70B) | llama-3.1-70b-instruct | 4 | 320 GB (4×80GB) | Large LLM with tensor parallelism |
| nemo-embedding | llama-3.2-nv-embedqa-1b-v2 | 1 | 8 GB | Vector embedding |
| nemo-rerank | llama-3.2-nv-rerankqa-1b-v2 | 1 | 8 GB | Document reranking |
| vss | Cosmos-Reason2-8B (int4_awq) | 1 | 22 GB | Vision-language model (quantized) |
| vss (fp16) | Cosmos-Reason2-8B | 2 | 44 GB (2×22GB) | Vision-language model (full precision) |

**Minimum GPU hardware:**
- **L40S** (46 GB VRAM) ✅ Supports all single-GPU components
- **A100 40GB** ✅ Supports all single-GPU components
- **A10G** (22 GB VRAM) ❌ Insufficient for Cosmos-Reason2-8B

**Total GPU count:**
- **4 GPUs minimum:** 1 LLM + 1 embedding + 1 reranking + 1 VLM
- **8 GPUs for 70B LLM:** 4 LLM + 1 embedding + 1 reranking + 2 VLM

## Known Issues and Gotchas

### Issue: Pod Stuck in Pending Due to GPU Taint

**Problem:** Pod remains `Pending` with event:
```
0/N nodes are available: N node(s) had untolerated taint {nvidia.com/gpu: present}
```

**Cause:** Missing toleration for GPU node taint.

**Solution:** Add toleration matching the node taint:
```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

### Issue: Insufficient GPUs Available

**Problem:** Pod stuck in `Pending` with event:
```
0/N nodes are available: N Insufficient nvidia.com/gpu
```

**Causes:**
1. Not enough GPU nodes in cluster
2. GPUs already allocated to other pods
3. Requesting more GPUs than available per node

**Debugging:**
```bash
# Check GPU availability across nodes
oc describe nodes -l nvidia.com/gpu.present=true | grep -A 5 "Allocated resources"

# Check which pods are using GPUs
oc get pods -A -o json | jq '.items[] | select(.spec.containers[].resources.limits."nvidia.com/gpu" != null) | {name: .metadata.name, namespace: .metadata.namespace, gpus: .spec.containers[].resources.limits."nvidia.com/gpu"}'
```

**Solutions:**
- Scale down or delete other GPU workloads
- Add more GPU nodes
- Reduce GPU request count

### Issue: GPU Not Detected in Pod

**Problem:** Pod schedules but can't access GPU (nvidia-smi fails).

**Causes:**
1. NVIDIA GPU Operator not installed or misconfigured
2. GPU device plugin not running
3. Pod missing resource request

**Debugging:**
```bash
# Check GPU operator pods
oc get pods -n nvidia-gpu-operator

# Verify device plugin is running
oc get pods -n nvidia-gpu-operator -l app=nvidia-device-plugin-daemonset

# Test GPU access in pod
oc exec <pod-name> -- nvidia-smi
```

**Solution:** Ensure GPU Operator is installed and healthy

### Issue: MIG (Multi-Instance GPU) Configuration

**Problem:** Cluster uses MIG but pods can't access GPU slices.

**Solution:** Use MIG resource names instead of `nvidia.com/gpu`:
```yaml
resources:
  limits:
    nvidia.com/mig-1g.5gb: 1  # MIG profile name
```

**MIG profile examples:**
- `nvidia.com/mig-1g.5gb`: 1 GPU instance, 5 GB memory
- `nvidia.com/mig-2g.10gb`: 2 GPU instances, 10 GB memory
- `nvidia.com/mig-3g.20gb`: 3 GPU instances, 20 GB memory

### Issue: Cluster-Specific Taints Not Documented

**Problem:** Pods still pending after adding `nvidia.com/gpu` toleration.

**Cause:** Cluster administrators added custom GPU taints.

**Solution:** Check actual taints and add them to values documentation:
```bash
oc describe nodes -l nvidia.com/gpu.present=true | grep Taints
```

Update values file comments:
```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  # Add cluster-specific tolerations as needed:
  # - key: <your-gpu-taint-key>
  #   operator: Equal
  #   value: "true"
  #   effect: NoSchedule
```

## Environment Variables for GPU Configuration

Some workloads need GPU-related environment variables:

### CUDA Visible Devices

Limit which GPUs are visible to the application:

```yaml
env:
  - name: CUDA_VISIBLE_DEVICES
    value: "0"  # Only GPU 0 visible
```

**When to Use:** Multi-GPU nodes where you want to isolate workloads

### NVIDIA Driver Capabilities

Enable specific GPU capabilities:

```yaml
env:
  - name: NVIDIA_DRIVER_CAPABILITIES
    value: "compute,utility"
```

**Common values:**
- `compute`: CUDA compute
- `utility`: nvidia-smi
- `graphics`: OpenGL
- `video`: Video encode/decode
- `all`: All capabilities

## Testing Notes

### Verify GPU Nodes
```bash
oc get nodes -l nvidia.com/gpu.present=true
```

### Check GPU Allocatable Resources
```bash
oc describe node <gpu-node> | grep nvidia.com/gpu
```

Expected output:
```
  nvidia.com/gpu: 1
  nvidia.com/gpu: 1
```

### Check GPU Taints
```bash
oc describe node <gpu-node> | grep Taints
```

### Verify Pod GPU Allocation
```bash
oc describe pod <pod-name> -n <namespace> | grep nvidia.com/gpu
```

Expected output:
```
    Limits:
      nvidia.com/gpu: 1
    Requests:
      nvidia.com/gpu: 1
```

### Test GPU Access in Pod
```bash
oc exec <pod-name> -n <namespace> -- nvidia-smi
```

Expected: GPU listing and utilization

### Check GPU Device Plugin
```bash
oc get pods -n nvidia-gpu-operator -l app=nvidia-device-plugin-daemonset
```

Should show DaemonSet pods running on all GPU nodes

## File Organization

```
deploy/helm/
└── <chart-name>/
    └── values.yaml
        ├── vss:
        │   ├── tolerations: [...]
        │   └── resources:
        │         limits:
        │           nvidia.com/gpu: 1
        └── nimOperator:
              <nim-name>:
                ├── tolerations: [...]
                └── resources:
                      limits:
                        nvidia.com/gpu: 1
```

## Best Practices

1. **Set Both Requests and Limits**: Always set `requests` equal to `limits` for GPUs
2. **Use Generic Tolerations**: Prefer `operator: Exists` over specific values for portability
3. **Document Cluster Taints**: Include placeholder comments for cluster-specific taints
4. **Verify GPU Requirements**: Test actual VRAM usage before documenting minimum requirements
5. **Single GPU Per Pod**: Default to 1 GPU unless tensor parallelism is required
6. **Node Selector as Optional**: Don't force specific GPU types unless necessary
7. **Test on Multiple Node Types**: Verify tolerations work across L40S, A100, H100, etc.
8. **Monitor GPU Utilization**: Use `nvidia-smi` in pods to verify GPU is actually used

## Conversion Checklist

When adding GPU support to a blueprint:

- [ ] Identify which components require GPUs
- [ ] Add `nvidia.com/gpu` resource requests/limits to each GPU component
- [ ] Add `nvidia.com/gpu` toleration to each GPU component
- [ ] Document minimum GPU requirements (type, VRAM, count)
- [ ] Add placeholder comments for cluster-specific taints
- [ ] Test scheduling on GPU nodes
- [ ] Verify GPU access in pods (`nvidia-smi`)
- [ ] Document total GPU count required for deployment
- [ ] Add nodeSelector configuration (optional, commented out)
- [ ] Test with different GPU node types (L40S, A100, H100)
- [ ] Update README with GPU prerequisites
- [ ] Add GPU verification commands to deployment guide
