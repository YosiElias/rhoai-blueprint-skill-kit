---
name: kibana-on-rhoai
description: Kibana deployment on RHOAI with OpenShift Routes and security contexts
metadata:
  type: component
components: [kibana]
deployment_types: [helm]
resource_types: [security-context, networking]
architecture: []
source_examples:
  - blueprint: "data-flywheel"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/data-flywheel"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-data-flywheel"
    notes: "Kibana with OpenShift Route for external access, TLS edge termination"
    approach: "A"
---

# Kibana on RHOAI

## Overview

Kibana is a visualization and user interface for Elasticsearch. This pattern shows how to deploy Kibana with OpenShift Routes for external access and security contexts.

## Conversion Pattern

### OPENSHIFT_MODE Conditional Support

Uses Helm template conditionals with `.Values.openshift.enabled` for security contexts and Routes.

### Deployment Type: Helm

Kibana is deployed as a Kubernetes Deployment with conditional security contexts and OpenShift Routes.

### Security Context Requirements

**Pod and container-level security contexts** - See [[elasticsearch-on-rhoai#Security Context Requirements]]

### Networking Configuration

#### Service Type Switching

The Service type changes based on OpenShift mode:

```yaml
# In service template
spec:
  selector:
    app: {{ .Values.kibana.fullnameOverride }}-deployment
  type: {{ include "data-flywheel.serviceType" . }}
  ports:
    - port: {{ .Values.kibana.service.port }}
```

**Helper function for service type**:
```yaml
{{/*
Determine service type based on OpenShift mode
*/}}
{{- define "data-flywheel.serviceType" -}}
{{- if .Values.openshift.enabled -}}
ClusterIP
{{- else -}}
NodePort
{{- end -}}
{{- end }}
```

**Why?**
- **OpenShift**: Uses ClusterIP because external access is via Routes
- **Standard Kubernetes**: Uses NodePort for external access without ingress

#### OpenShift Route

Create a separate Route resource for external access:

**File**: `templates/kibana-route.yaml`
```yaml
{{- if and .Values.openshift.enabled .Values.openshift.routes.enabled .Values.openshift.routes.kibana.enabled .Values.kibana.enabled }}
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ .Values.kibana.fullnameOverride }}-route
  labels:
    app: {{ .Values.kibana.fullnameOverride }}-deployment
spec:
  {{- if .Values.openshift.routes.kibana.host }}
  host: {{ .Values.openshift.routes.kibana.host }}
  {{- end }}
  to:
    kind: Service
    name: {{ .Values.kibana.fullnameOverride }}-service
    weight: 100
  port:
    targetPort: {{ .Values.kibana.service.port }}
  {{- if .Values.openshift.routes.kibana.tls.enabled }}
  tls:
    termination: {{ .Values.openshift.routes.kibana.tls.termination }}
    insecureEdgeTerminationPolicy: {{ .Values.openshift.routes.kibana.tls.insecureEdgeTerminationPolicy }}
  {{- end }}
  wildcardPolicy: None
{{- end }}
```

**Values configuration**:
```yaml
openshift:
  enabled: false
  routes:
    enabled: true
    kibana:
      enabled: true
      host: ""  # Auto-generated if empty
      tls:
        enabled: true
        termination: edge
        insecureEdgeTerminationPolicy: Redirect
```

**Conditional logic explained**:
- Only create Route if all of these are true:
  - `openshift.enabled=true`
  - `openshift.routes.enabled=true`
  - `openshift.routes.kibana.enabled=true`
  - `kibana.enabled=true` (the component itself is enabled)

### Container Image

Uses standard Kibana image:
```yaml
kibana:
  image:
    repository: "docker.elastic.co/kibana/kibana"
    tag: "8.8.0"
```

### Environment Variables

Kibana needs to know the Elasticsearch service endpoint:
```yaml
env:
  - name: ELASTICSEARCH_HOSTS
    value: "http://df-elasticsearch-service:9200"
```

## Known Issues and Gotchas

### Issue: Route host auto-generation
- **Behavior**: If `host` is left empty, OpenShift auto-generates a hostname based on route name, namespace, and cluster domain
- **Example**: `df-kibana-route-data-flywheel.apps.cluster.example.com`
- **Recommendation**: For production, specify explicit hostnames

### Issue: TLS edge termination
- **Behavior**: TLS is terminated at the OpenShift router, backend communication is HTTP
- **Why**: Kibana runs on HTTP internally, OpenShift router handles HTTPS
- **Alternative**: Use `termination: reencrypt` if Kibana is configured with TLS

## Dependencies

- **Elasticsearch**: Kibana requires Elasticsearch to be running and accessible
  - See [[elasticsearch-on-rhoai]]

## Testing Notes

Verify Kibana is running and accessible:
```bash
# Check pod status
oc get pods -n $NAMESPACE | grep kibana

# Get Route URL
KIBANA_URL=$(oc get route df-kibana-route -n $NAMESPACE -o jsonpath='{.spec.host}')
echo "Kibana UI: https://$KIBANA_URL"

# Test Kibana health
curl -k "https://$KIBANA_URL/api/status"
```

## Related Patterns

- [[elasticsearch-on-rhoai]] - Backend for Kibana
- [[mlflow-on-rhoai]] - Similar web UI with Routes
- [[networking-routes-ingress]] - General Route patterns for OpenShift
