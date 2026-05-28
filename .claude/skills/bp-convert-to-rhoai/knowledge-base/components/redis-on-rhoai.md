---
name: redis-on-rhoai
description: Redis deployment on RHOAI with OpenShift-compatible security contexts and init containers
metadata:
  type: component
components: [redis]
deployment_types: [helm]
resource_types: [storage, security-context]
architecture: []
source_examples:
  - blueprint: "data-flywheel"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/data-flywheel"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-data-flywheel"
    notes: "Redis with conditional OpenShift support, restricted SCC compliance"
    approach: "A"
---

# Redis on RHOAI

## Overview

Redis is an in-memory data store commonly used as a message broker for Celery task queues in AI/ML pipelines. This pattern shows how to deploy Redis with OpenShift conditional support using Helm templates.

## Conversion Pattern

### OPENSHIFT_MODE Conditional Support

Uses Helm template conditionals with `.Values.openshift.enabled` to switch between standard Kubernetes and OpenShift configurations.

### Deployment Type: Helm

Redis is deployed as a Kubernetes Deployment with conditional security contexts and init containers.

### Security Context Requirements

**Pod-level security context** (when `openshift.enabled=true`):
```yaml
spec:
  automountServiceAccountToken: false
  {{- if .Values.openshift.enabled }}
  securityContext:
    {{- include "data-flywheel.podSecurityContext" . | nindent 8 }}
  {{- end }}
```

**Container-level security context** (when `openshift.enabled=true`):
```yaml
containers:
  - name: redis
    image: "{{ .Values.redis.image.repository }}:{{ .Values.redis.image.tag }}"
    imagePullPolicy: Always
    {{- if .Values.openshift.enabled }}
    securityContext:
      {{- include "data-flywheel.containerSecurityContext" . | nindent 12 }}
    {{- end }}
```

**Helper function definitions** - See [[mongodb-on-rhoai#Helper function definitions]]

**Values configuration**:
```yaml
openshift:
  enabled: false
  securityContext:
    pod:
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
    container:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      capabilities:
        drop:
          - ALL
```

### Storage Configuration

Uses **emptyDir** volumes for ephemeral storage with conditional init containers:

**When OpenShift enabled**:
```yaml
initContainers:
  {{- if .Values.openshift.enabled }}
  - name: create-redis-data-folder
    image: {{ .Values.openshift.initContainer.image }}
    command: ['sh', '-c', 'mkdir -p /data']
    volumeMounts:
      - name: redis-data-volume
        mountPath: /data
    securityContext:
      {{- include "data-flywheel.containerSecurityContext" . | nindent 12 }}
  {{- else }}
  - name: create-redis-data-folder
    image: busybox:1.28
    command: ['sh', '-c', 'mkdir -p /data && chmod 755 /data']
    volumeMounts:
      - name: redis-data-volume
        mountPath: /data
  {{- end }}
```

**Volume mount and volume**:
```yaml
containers:
  - name: redis
    # ... other config ...
    volumeMounts:
      - name: redis-data-volume
        mountPath: /data
volumes:
  - name: redis-data-volume
    emptyDir: {}
```

**Why different init containers?**
- **OpenShift**: Uses Red Hat UBI minimal image (`registry.access.redhat.com/ubi8/ubi-minimal:latest`) and cannot run `chmod` due to restricted SCC
- **Standard Kubernetes**: Uses `busybox:1.28` which can run `chmod` for explicit permissions

### Container Image

Uses standard Redis from container registry:
```yaml
redis:
  image:
    repository: "redis"
    tag: "latest"
```

## Known Issues and Gotchas

### Issue: chmod fails in restricted SCC
- **Problem**: Init containers cannot run `chmod` when using restricted-v2 SCC
- **Solution**: Remove `chmod` command in OpenShift mode; rely on OpenShift's automatic UID/GID assignment from namespace range

### Issue: Redis data persistence
- **Problem**: Using emptyDir means Redis data is lost when pod restarts
- **Solution**: For production, consider using PersistentVolumeClaim instead of emptyDir. This pattern uses emptyDir for task queue scenarios where data loss is acceptable (Celery tasks can be retried).

## Dependencies

None - Redis is a standalone component.

## Testing Notes

Verify Redis is running and accessible:
```bash
# Check pod status
oc get pods -n $NAMESPACE | grep redis

# Test Redis connection from another pod
oc exec -n $NAMESPACE deployment/df-api -- redis-cli -h df-redis-service ping
# Expected: PONG
```

## Related Patterns

- [[mongodb-on-rhoai]] - Similar stateful service with conditional OpenShift support
- [[security-contexts-scc]] - Security context patterns for OpenShift
