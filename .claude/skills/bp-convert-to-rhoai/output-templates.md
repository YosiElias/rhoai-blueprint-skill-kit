# Output Templates for RHOAI Conversion

This document contains templates for documentation generated during blueprint conversion. Read the appropriate template when generating each output document.

## TEST-PLAN.md Template

```markdown
# RHOAI Conversion Test Plan

## Prerequisites
- OpenShift cluster 4.x+ with GPU operator (if GPU workloads)
- `oc` CLI logged in with cluster-admin
- Helm v3+ (if Helm deployment)
- NGC API key (if NVIDIA images)

## Environment Setup
\```bash
export NAMESPACE="<blueprint-name>-test"

# For oc apply deployments:
export OPENSHIFT_MODE=true

# For Helm deployments (set during helm install instead):
# --set openshiftMode=true

# Add other required env vars based on blueprint
\```

## Deployment Steps

### For Helm Deployments:
\```bash
helm install <release-name> ./ \\
  --namespace $NAMESPACE \\
  --create-namespace \\
  --set openshiftMode=true \\
  --set serverHost=<your-openshift-route-hostname>
\```

### For OC Apply Deployments:
\```bash
export OPENSHIFT_MODE=true
oc create namespace $NAMESPACE
oc apply -f manifests/ -n $NAMESPACE
\```

## Verification Checklist
- [ ] All pods reach Running state
  \```bash
  oc get pods -n $NAMESPACE
  \```

- [ ] GPU resources allocated correctly (if applicable)
  \```bash
  oc describe pod <gpu-pod> -n $NAMESPACE | grep -A 5 "Limits"
  \```

- [ ] PVCs bound successfully
  \```bash
  oc get pvc -n $NAMESPACE
  \```

- [ ] Routes accessible
  \```bash
  oc get routes -n $NAMESPACE
  curl http://<route-host>/health
  \```

- [ ] Service-to-service communication works
  [Service-specific tests based on blueprint]

- [ ] Original functionality preserved
  \```bash
  # Test with RHOAI mode disabled:
  # For Helm: helm install ... --set openshiftMode=false
  # For oc apply: unset OPENSHIFT_MODE; oc apply ...
  \```

## Component-Specific Tests
[Generated based on blueprint components - add specific validation for each service]

## Rollback Plan
\```bash
# For Helm:
helm uninstall <release-name> -n $NAMESPACE

# For OC Apply:
oc delete -f manifests/ -n $NAMESPACE

# Clean up namespace
oc delete namespace $NAMESPACE
\```

## Known Issues
[List any known issues or workarounds discovered during conversion]
```

---

## RHOAI-CONVERSION.md Template

```markdown
# RHOAI Conversion Summary

## Overview
This NVIDIA Blueprint has been adapted to run on Red Hat OpenShift AI (RHOAI) with minimal invasive changes.

## Conditional RHOAI Support

### For Helm Deployments
- Added `openshiftMode` flag in values.yaml (default: false)
- Original behavior preserved when openshiftMode=false
- RHOAI-specific configuration activated when openshiftMode=true

**Usage:**
\```bash
# Original deployment (non-RHOAI):
helm install myapp ./ --set openshiftMode=false

# RHOAI deployment:
helm install myapp ./ --set openshiftMode=true
\```

### For OC Apply Deployments
- Added checks for `OPENSHIFT_MODE` environment variable
- Original behavior when OPENSHIFT_MODE is unset or false
- RHOAI-specific configuration when OPENSHIFT_MODE=true

**Usage:**
\```bash
# Original deployment (non-RHOAI):
oc apply -f manifests/

# RHOAI deployment:
export OPENSHIFT_MODE=true
oc apply -f manifests/
\```

## Components Modified

[For each component, document what changed]

### <Component Name 1>
- **Type**: [Inference server|Database|Cache|etc.]
- **Security Context**: Added anyuid SCC requirement, runAsUser: 0
- **Storage**: PVC with ReadWriteOnce access mode, 50Gi
- **GPU**: nvidia.com/gpu: 1, node selectors + tolerations
- **Networking**: OpenShift Route at /component-path
- **Configuration**: [Other RHOAI-specific changes]

### <Component Name 2>
...

## Files Changed

### Modified
- `values.yaml` (Helm): Added RHOAI configuration flags
- `templates/deployments/*.yaml` (Helm): Added conditional security contexts, GPU allocation
- `docker-compose.yaml`: Added OPENSHIFT_MODE environment checks (if applicable)

### Created (RHOAI-specific)
- `templates/routes.yaml` (Helm) or `manifests/route.yaml`: OpenShift Routes for external access
- [Other new files if absolutely necessary]

## Deployment Method
- **[Helm Chart|OC Apply]**: [Reason for choice]
- All resources are [Helm-managed templates|standalone manifests]

## Resource Requirements
- **GPU**: <count> x <type> (<total VRAM>)
- **CPU**: <total requests> / <total limits>
- **Memory**: <total requests> / <total limits>
- **Storage**: <total PVC size>

## Knowledge Sources Applied
[List knowledge files that informed this conversion]

- `components/triton-on-rhoai.md` (Approach A - dedicated node pool)
- `components/milvus-on-rhoai.md`
- `resource-patterns/gpu-allocation-openshift.md`
- `resource-patterns/storage-pvc-patterns.md`
- `resource-patterns/networking-routes-ingress.md`

## User Decisions Made
[Document choices the user made during conversion]

- **Model deployment strategy**: <Local|NVIDIA API|Hybrid>
  - Rationale: [Why this choice]
- **GPU allocation approach**: <Dedicated node pool|Shared pool>
  - Rationale: [Why this choice]
- [Other decisions]

## Testing
See `TEST-PLAN.md` for detailed verification steps and component-specific tests.

## Known Limitations
[Any limitations or manual steps still required]

## Support
For issues or questions:
1. Check `TEST-PLAN.md` for troubleshooting steps
2. Review knowledge base files for component-specific guidance
3. Consult Red Hat OpenShift AI documentation
4. Check blueprint-specific documentation in original repo
```

---

## Conversion Summary Report Template

Print this summary to user at the end:

```
=== RHOAI Conversion Complete ===

Blueprint: <name>
Architecture: <architecture-type>
Deployment Method: <Helm|OC Apply>
RHOAI Mode Toggle: <openshiftMode (Helm)|OPENSHIFT_MODE (env var)>

Components Converted: <count>
- <component-1>: <brief what changed>
- <component-2>: <brief what changed>
  ...

Patterns Applied:
- GPU allocation: <pattern used>
- Storage: <pattern used>
- Security: <pattern used>
- Networking: <pattern used>

User Decisions:
- Model deployment: <choice and rationale>
- GPU strategy: <choice>
- [Other decisions]

Files Modified: <count>
[List key modified files]

Files Created: <count>
[List new files]

Knowledge Sources Used:
- <knowledge-file-1>
- <knowledge-file-2>
- ...

Context7 Queries Made: <count>
[If any queries were made to Red Hat docs]

Next Steps:
1. Review modified files for correctness
2. Follow TEST-PLAN.md for deployment verification
3. Test with RHOAI mode enabled (openshiftMode=true or OPENSHIFT_MODE=true)
4. Validate original deployment still works with RHOAI mode disabled
5. Document any additional manual customizations needed

Estimated Conversion Coverage:
- Pattern-matched: ~80%
- Custom decisions: ~20%
- Manual review recommended: [specific areas if any]
```
