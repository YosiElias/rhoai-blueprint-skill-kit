---
name: elasticsearch-on-rhoai
description: Elasticsearch deployment on RHOAI with OpenShift-compatible security contexts
metadata:
  type: component
components: [elasticsearch]
deployment_types: [helm]
resource_types: [security-context]
architecture: []
source_examples:
  - blueprint: "data-flywheel"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/data-flywheel"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-data-flywheel"
    notes: "Elasticsearch with conditional OpenShift support, restricted SCC compliance, no init containers needed"
    approach: "A"
---

# Elasticsearch on RHOAI

## Overview

Elasticsearch is a distributed search and analytics engine commonly used for logging and data storage in AI/ML pipelines. This pattern shows how to deploy Elasticsearch with OpenShift conditional support using Helm templates.

## Conversion Pattern

### OPENSHIFT_MODE Conditional Support

Uses Helm template conditionals with `.Values.openshift.enabled` to switch between standard Kubernetes and OpenShift configurations.

### Deployment Type: Helm

Elasticsearch is deployed as a Kubernetes Deployment with conditional security contexts.

### Security Context Requirements

**Pod-level security context** (when `openshift.enabled=true`):
```yaml
spec:
  automountServiceAccountToken: false
  {{- if .Values.openshift.enabled }}
  securityContext:
    {{- include "data-flywheel.podSecurityContext" . | nindent 8 }}
  {{- end }}
  containers:
    - name: elasticsearch
      image: "{{ .Values.elasticsearch.image.repository }}:{{ .Values.elasticsearch.image.tag }}"
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

**No explicit storage volumes** - Elasticsearch uses container filesystem for ephemeral data.

**No init containers needed** - Unlike MongoDB and Redis, Elasticsearch does not require data folder pre-creation.

### Container Image

Uses standard Elasticsearch image:
```yaml
elasticsearch:
  image:
    repository: "docker.elastic.co/elasticsearch/elasticsearch"
    tag: "8.8.0"
```

### Environment Variables

Typical Elasticsearch configuration for single-node development:
```yaml
env:
  - name: discovery.type
    value: "single-node"
  - name: ES_JAVA_OPTS
    value: "-Xms2g -Xmx2g"
```

## Known Issues and Gotchas

### Issue: Elasticsearch requires vm.max_map_count setting
- **Problem**: Elasticsearch requires `vm.max_map_count` to be at least 262144 on the host
- **Solution**: This is a cluster-level setting that must be configured by the cluster administrator:
  ```bash
  # On OpenShift nodes (requires cluster admin)
  oc debug node/<node-name>
  chroot /host
  sysctl -w vm.max_map_count=262144
  ```

### Issue: Memory limits and Java heap
- **Problem**: Elasticsearch JVM heap must be sized appropriately relative to container memory limits
- **Solution**: Set `ES_JAVA_OPTS` to use roughly 50% of container memory limit:
  ```yaml
  resources:
    limits:
      memory: "4Gi"
  env:
    - name: ES_JAVA_OPTS
      value: "-Xms2g -Xmx2g"  # 50% of 4Gi
  ```

## Dependencies

None - Elasticsearch is a standalone component.

## Testing Notes

Verify Elasticsearch is running and accessible:
```bash
# Check pod status
oc get pods -n $NAMESPACE | grep elasticsearch

# Test Elasticsearch health from another pod
oc exec -n $NAMESPACE deployment/df-api -- curl -s http://df-elasticsearch-service:9200/_cluster/health
```

## Related Patterns

- [[kibana-on-rhoai]] - Visualization UI for Elasticsearch
- [[security-contexts-scc]] - Security context patterns for OpenShift
