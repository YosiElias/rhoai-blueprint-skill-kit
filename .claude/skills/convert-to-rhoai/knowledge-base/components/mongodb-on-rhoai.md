---
name: mongodb-on-rhoai
description: MongoDB deployment on RHOAI with OpenShift-compatible security contexts and init containers
metadata:
  type: component
components: [mongodb]
deployment_types: [helm]
resource_types: [storage, security-context]
architecture: []
source_examples:
  - blueprint: "data-flywheel"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/data-flywheel"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-data-flywheel"
    notes: "MongoDB with conditional OpenShift support, restricted SCC compliance"
    approach: "A"
---

# MongoDB on RHOAI

## Overview

MongoDB is a NoSQL document database commonly used for metadata storage in AI/ML pipelines. This pattern shows how to deploy MongoDB with OpenShift conditional support using Helm templates.

## Conversion Pattern

### OPENSHIFT_MODE Conditional Support

Uses Helm template conditionals with `.Values.openshift.enabled` to switch between standard Kubernetes and OpenShift configurations.

### Deployment Type: Helm

MongoDB is deployed as a Kubernetes Deployment with conditional security contexts and init containers.

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
  - name: mongodb
    image: "{{ .Values.mongodb.image.repository }}:{{ .Values.mongodb.image.tag }}"
    imagePullPolicy: Always
    {{- if .Values.openshift.enabled }}
    securityContext:
      {{- include "data-flywheel.containerSecurityContext" . | nindent 12 }}
    {{- end }}
```

**Helper function definitions** (in `_helpers.tpl`):
```yaml
{{/*
Generate pod-level security context for OpenShift
*/}}
{{- define "data-flywheel.podSecurityContext" -}}
{{- if .Values.openshift.enabled }}
runAsNonRoot: true
seccompProfile:
  type: {{ .Values.openshift.securityContext.pod.seccompProfile.type }}
{{- end }}
{{- end }}

{{/*
Generate container-level security context for OpenShift
*/}}
{{- define "data-flywheel.containerSecurityContext" -}}
{{- if .Values.openshift.enabled }}
allowPrivilegeEscalation: {{ .Values.openshift.securityContext.container.allowPrivilegeEscalation }}
runAsNonRoot: {{ .Values.openshift.securityContext.container.runAsNonRoot }}
capabilities:
  drop:
    {{- range .Values.openshift.securityContext.container.capabilities.drop }}
    - {{ . }}
    {{- end }}
{{- end }}
{{- end }}
```

**Values configuration**:
```yaml
openshift:
  enabled: false # Set to true to enable OpenShift-compatible deployment
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
  - name: create-mongodb-data-folder
    image: {{ .Values.openshift.initContainer.image }}
    command: ['sh', '-c', 'mkdir -p /data/db']
    volumeMounts:
      - name: mongodb-data-volume
        mountPath: /data/db
    securityContext:
      {{- include "data-flywheel.containerSecurityContext" . | nindent 12 }}
  {{- else }}
  - name: create-mongodb-data-folder
    image: busybox:1.28
    command: ['sh', '-c', 'mkdir -p /data/db && chmod 755 /data/db']
    volumeMounts:
      - name: mongodb-data-volume
        mountPath: /data/db
  {{- end }}
```

**Volume mount and volume**:
```yaml
containers:
  - name: mongodb
    # ... other config ...
    volumeMounts:
      - name: mongodb-data-volume
        mountPath: /data/db
volumes:
  - name: mongodb-data-volume
    emptyDir: {}
```

**Why different init containers?**
- **OpenShift**: Uses Red Hat UBI minimal image (`registry.access.redhat.com/ubi8/ubi-minimal:latest`) and cannot run `chmod` due to restricted SCC
- **Standard Kubernetes**: Uses `busybox:1.28` which can run `chmod` for explicit permissions

### Container Image

Uses standard MongoDB from Docker Hub:
```yaml
mongodb:
  image:
    repository: "docker.io/mongo"
    tag: "latest"
```

**Why Docker Hub instead of NGC?**
- MongoDB is not an NVIDIA-specific service
- Docker Hub images are well-tested and OpenShift-compatible
- No GPU requirements

## Known Issues and Gotchas

### Issue: chmod fails in restricted SCC
- **Problem**: Init containers cannot run `chmod` when using restricted-v2 SCC
- **Solution**: Remove `chmod` command in OpenShift mode; rely on OpenShift's automatic UID/GID assignment from namespace range

### Issue: fsGroup conflicts with restricted-v2 SCC
- **Problem**: Explicitly setting `fsGroup` in pod security context causes SCC violations
- **Solution**: Do not set `fsGroup` explicitly; OpenShift automatically assigns it from the namespace UID/GID range

## Dependencies

None - MongoDB is a standalone component.

## Testing Notes

Verify MongoDB is running and accessible:
```bash
# Check pod status
oc get pods -n $NAMESPACE | grep mongodb

# Test MongoDB connection from another pod
oc exec -n $NAMESPACE deployment/df-api -- mongosh --host df-mongodb-service --eval "db.version()"
```

## Related Patterns

- [[redis-on-rhoai]] - Similar stateful service with conditional OpenShift support
- [[security-contexts-scc]] - Security context patterns for OpenShift
