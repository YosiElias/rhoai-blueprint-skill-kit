# Dynamic Reasoning Guardrails

This document defines the concern areas that should be investigated during RHOAI blueprint conversion. These are **not** a checklist to mechanically fill out, but a mental framework to ensure critical aspects aren't overlooked while reasoning dynamically.

## Purpose

When converting a blueprint, questions should emerge organically from analysis. However, certain concerns are easy to miss without explicit attention. These guardrails ensure comprehensive coverage.

## How to Use

As you reason about the conversion:
1. Think freely - let questions emerge naturally from blueprint analysis
2. Periodically check: "Have I considered [concern area]?"
3. If not yet addressed, reason about it explicitly
4. Don't force irrelevant concerns (e.g., GPU if blueprint has no GPU workloads)

## Concern Areas

### 1. Resource Allocation
**What to consider:**
- CPU requests and limits for each service
- Memory requirements (especially for workers, ML inference)
- Storage capacity needs
- GPU requirements (type, count, VRAM)

**Key questions:**
- Does this component need more resources than default limits?
- Are there worker processes that could OOM with insufficient memory?
- What GPU resources are needed (if any)?

**Where to look:**
- docker-compose `deploy.resources` sections
- Known resource-intensive services (Celery workers, ML inference, Java services)
- GPU requirements in deployment specs

---

### 2. Security Contexts and SCCs
**What to consider:**
- OpenShift Security Context Constraints (SCC) requirements
- runAsUser, runAsGroup, fsGroup settings
- Capabilities needed
- Whether containers can run as non-root

**Key questions:**
- Do NVIDIA containers need to run as root (typically yes → anyuid SCC)?
- Are there file ownership issues that need fsGroup?
- What's the minimum SCC required (avoid privileged if possible)?

**Where to look:**
- SecurityContext in pod specs
- Container image documentation (NVIDIA images typically need root)
- Volume mount permissions

---

### 3. Networking
**What to consider:**
- OpenShift Routes vs Kubernetes Ingress
- Service discovery and DNS
- Path-based routing strategies
- WebSocket timeouts
- TLS/SSL termination

**Key questions:**
- Which services need external access (Routes)?
- How should services find each other (Kubernetes DNS, service mesh)?
- Are there WebSocket connections needing longer timeouts?
- Should traffic be path-based on single host or multiple hosts?

**Where to look:**
- docker-compose `ports` mappings
- Service inter-dependencies
- WebSocket or streaming services
- Frontend/UI services

---

### 4. Persistent Storage
**What to consider:**
- PersistentVolumeClaim requirements
- Access modes (ReadWriteOnce vs ReadWriteMany)
- StorageClass selection
- Volume mount patterns (subPath isolation)
- Ephemeral vs persistent data

**Key questions:**
- Which data must persist across pod restarts?
- Can multiple pods access the same volume simultaneously (RWX needed)?
- What's the total storage capacity required?
- Should volumes be shared with subPath isolation?

**Where to look:**
- docker-compose `volumes` sections
- Stateful services (databases, caches)
- Services that write logs or artifacts

---

### 5. Inter-Service Dependencies
**What to consider:**
- Service initialization order
- Health check requirements
- Dependency waiting (init containers, readiness probes)
- Service mesh integration
- mTLS requirements

**Key questions:**
- Does service A depend on service B being ready first?
- Are there circular dependencies to handle?
- Should we use init containers or readiness probes?
- Is service mesh (Istio) needed for service-to-service communication?

**Where to look:**
- docker-compose `depends_on` clauses
- Application startup logic
- Health check endpoints

---

### 6. Initialization Order
**What to consider:**
- Database schema initialization
- Secret/config prerequisites
- Service dependencies
- One-time setup jobs

**Key questions:**
- Do databases need schema migrations before app starts?
- Are there bootstrap scripts that should run once?
- Should we use Kubernetes Jobs for init tasks?

**Where to look:**
- docker-compose `entrypoint` and `command` overrides
- Init scripts in repository
- Database migration tools

---

### 7. Secrets and Config Management
**What to consider:**
- Sensitive vs non-sensitive configuration
- Secret injection methods (env vars, files, volumes)
- ConfigMap usage
- OPENSHIFT_MODE conditional configuration
- Credential consistency with docker-compose defaults

**Key questions:**
- What values are sensitive (passwords, API keys, tokens)?
- Should secrets be injected as env vars or mounted as files?
- Are docker-compose default credentials hardcoded in app (keep them)?
- How should OPENSHIFT_MODE conditionals be structured?

**Where to look:**
- docker-compose `environment` sections
- Variables with PASSWORD, SECRET, TOKEN, KEY in names
- Mounted secret files

---

### 8. Image Registries and Pull Secrets
**What to consider:**
- Private vs public image registries
- NGC (NVIDIA GPU Cloud) pull secrets
- ImagePullSecrets configuration
- Image tag/version consistency

**Key questions:**
- Are NVIDIA images from nvcr.io (needs NGC secret)?
- Do custom images need registry authentication?
- Should we use specific tags or "latest"?

**Where to look:**
- docker-compose `image` fields
- Images from nvcr.io, private registries
- Build contexts (custom images)

---

### 9. Health Checks and Probes
**What to consider:**
- Liveness probes (restart unhealthy pods)
- Readiness probes (route traffic when ready)
- Startup probes (slow-starting services)
- Probe timeouts and thresholds

**Key questions:**
- How do we know if a service is healthy?
- When is it ready to receive traffic?
- Are there slow-starting services needing startup probes?

**Where to look:**
- docker-compose `healthcheck` sections
- Known health endpoints (/health, /healthz, /ready)
- Application documentation

---

### 10. Resource Quotas and Limits
**What to consider:**
- Namespace resource quotas
- LimitRanges
- Pod resource requests and limits balance
- QoS classes (Guaranteed, Burstable, BestEffort)

**Key questions:**
- Are resources properly requested (not just limited)?
- Will the cluster have capacity for these workloads?
- Should we use Guaranteed QoS for critical services?

**Where to look:**
- Cluster constraints (if known)
- Critical vs non-critical services
- Resource-intensive workloads

---

## Additional Concerns (Context-Specific)

### GPU Workloads
- Node selectors for GPU nodes
- Tolerations for GPU taints
- nvidia.com/gpu resource specification
- Shared vs dedicated GPU pools

### Model Deployment
- Local deployment vs NVIDIA API
- Model size and GPU VRAM requirements
- NIM model vs custom model serving
- API key management for hosted models

### Multi-Tenancy
- Namespace isolation
- NetworkPolicies
- RBAC requirements
- Resource quota per tenant

---

## Dynamic Reasoning Example

```
Analyzing blueprint...
  ↓ Found: Triton Inference Server in docker-compose
  
Question emerges: "How should Triton be deployed on RHOAI?"
  ↓ Check knowledge base: triton-on-rhoai.md
  ↓ Answer: Needs GPU, anyuid SCC, dedicated node pool pattern
  
Guardrail check: "Have I considered GPU allocation?" ✓ Yes
Guardrail check: "Have I considered security contexts?" ✓ Yes, anyuid SCC
  
Question emerges: "This Triton connects to Milvus - are there networking requirements?"
  ↓ Check guardrails: Networking ✓ Inter-service dependencies ✓
  ↓ Check knowledge base: triton-milvus-integration.md
  ↓ Answer: Service mesh config + init order requirements
  
Continue reasoning...
```

## When to Stop Checking Guardrails

Once you've reasoned about all applicable concerns:
- Concerns that don't apply to this blueprint can be skipped
- If a concern was implicitly handled during reasoning, that counts
- Don't force concerns that are truly irrelevant

## Self-Check Before Generating Modifications

Before generating RHOAI modifications, quickly verify:
- [ ] All relevant guardrails considered
- [ ] Decisions documented (implicitly or explicitly)
- [ ] Edge cases identified
- [ ] User decision points identified

If any guardrail feels unaddressed, reason about it explicitly before proceeding.
