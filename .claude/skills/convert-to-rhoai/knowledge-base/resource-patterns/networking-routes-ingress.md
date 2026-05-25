---
type: resource-pattern
components: []
deployment_types: [helm]
resource_types: [networking]
architecture: []
source_examples:
  - blueprint: "video-search-and-summarization"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-video-search-and-summarization"
    notes: "Demonstrates OpenShift Route creation with TLS edge termination for web UI access"
    approach: "A"
---

# OpenShift Routes for External Access

## Overview

OpenShift Routes provide external access to services, replacing Kubernetes Ingress or kubectl port-forward patterns used in upstream NVIDIA Blueprints. Routes support TLS termination, path-based routing, and integration with OpenShift's built-in load balancer.

## When to Use

- Exposing web UIs or APIs to external users
- Replacing `kubectl port-forward` or NodePort services in upstream blueprints
- When you need TLS termination without managing certificates manually
- Production-grade ingress with OpenShift's integrated routing layer

## Upstream Patterns vs. OpenShift

| Upstream Pattern | OpenShift Alternative | Benefits |
|------------------|----------------------|----------|
| `kubectl port-forward` | OpenShift Route | Persistent, production-ready |
| Kubernetes Ingress | OpenShift Route | Native OpenShift integration |
| NodePort Service | OpenShift Route | No need to manage node firewall rules |
| LoadBalancer Service | OpenShift Route | Uses cluster ingress controller |

## Route Creation Pattern

### 1. Basic Route Resource

**Example from openshift.yaml:**
```yaml
{{- if .Values.openshift.route.enabled }}
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ .Release.Name }}-ui
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  {{- if .Values.openshift.route.host }}
  host: {{ .Values.openshift.route.host | quote }}
  {{- end }}
  to:
    kind: Service
    name: vss-service
    weight: 100
  port:
    targetPort: webui  # Named port from Service
  tls:
    termination: {{ .Values.openshift.route.tls.termination }}
    insecureEdgeTerminationPolicy: {{ .Values.openshift.route.tls.insecureEdgeTerminationPolicy }}
  wildcardPolicy: None
{{- end }}
```

### 2. Values Configuration

**values-openshift.yaml:**
```yaml
openshift:
  enabled: true
  route:
    enabled: true
    host: ""  # Auto-generated if empty
    tls:
      termination: edge
      insecureEdgeTerminationPolicy: Redirect
```

### 3. Conditional Rendering

Gate Route creation behind `openshift.enabled` flag:

```yaml
{{- if .Values.openshift.enabled }}
{{- if .Values.openshift.route.enabled }}
# ... Route resource
{{- end }}
{{- end }}
```

## TLS Termination Options

### Edge Termination (Recommended)

**What:** TLS terminates at the router, traffic to backend is HTTP.

**Configuration:**
```yaml
tls:
  termination: edge
  insecureEdgeTerminationPolicy: Redirect  # HTTP → HTTPS redirect
```

**Benefits:**
- Simple configuration
- OpenShift manages certificates
- Backend doesn't need TLS support

**Use When:** Backend service only supports HTTP (most blueprint UIs)

### Passthrough Termination

**What:** TLS connection passes through router to backend unchanged.

**Configuration:**
```yaml
tls:
  termination: passthrough
```

**Benefits:**
- End-to-end encryption
- Backend controls certificate

**Use When:** Backend already has TLS configured and you want end-to-end encryption

**Note:** Requires backend service to handle TLS on the named port

### Re-encrypt Termination

**What:** TLS terminates at router, then re-encrypted to backend.

**Configuration:**
```yaml
tls:
  termination: reencrypt
  destinationCACertificate: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

**Use When:** Need end-to-end encryption but want router to inspect traffic

## Host Configuration

### Auto-Generated Host (Recommended for Dev)

Leave `host` empty and OpenShift generates: `<route-name>-<namespace>.<cluster-domain>`

**Configuration:**
```yaml
openshift:
  route:
    host: ""
```

**Getting the URL:**
```bash
oc get route <release-name>-ui -n <namespace> -o jsonpath='{.spec.host}'
```

### Custom Host (Production)

Specify custom hostname (requires DNS configuration):

**Configuration:**
```yaml
openshift:
  route:
    host: "vss.example.com"
```

**Requirements:**
- DNS record pointing to cluster ingress
- Certificate trust (edge termination uses cluster wildcard cert)

## Target Service Configuration

### Named Port Reference

Routes should reference Service ports by name, not number:

**Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: vss-service
spec:
  ports:
    - name: webui
      port: 3000
      targetPort: 3000
      protocol: TCP
```

**Route:**
```yaml
spec:
  port:
    targetPort: webui  # References port name
```

**Benefits:**
- Port numbers can change without breaking Route
- More readable

### Port Number Reference

Alternatively, reference by port number:

```yaml
spec:
  port:
    targetPort: 3000
```

## Path-Based Routing

Route to different backends based on URL path:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ .Release.Name }}-api
spec:
  path: /api
  to:
    kind: Service
    name: api-service
  tls:
    termination: edge
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ .Release.Name }}-ui
spec:
  path: /
  to:
    kind: Service
    name: ui-service
  tls:
    termination: edge
```

**Note:** More specific paths take precedence

## Multiple Routes for One Service

Create multiple routes for different purposes:

**Example:**
```yaml
# Internal route (short hostname)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ .Release.Name }}-internal
spec:
  host: vss-internal.apps.cluster.local
  to:
    kind: Service
    name: vss-service
---
# External route (custom domain)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ .Release.Name }}-external
spec:
  host: vss.example.com
  to:
    kind: Service
    name: vss-service
  tls:
    termination: edge
```

## Known Issues and Gotchas

### Issue: HTTP 503 Service Unavailable

**Problem:** Route returns 503 even though pods are running.

**Causes:**
1. Service selector doesn't match pods
2. Service port/targetPort mismatch
3. Backend pods not ready
4. Service doesn't exist

**Debugging:**
```bash
# Check route status
oc describe route <route-name> -n <namespace>

# Check service endpoints
oc get endpoints <service-name> -n <namespace>

# Verify pods are ready
oc get pods -n <namespace> -l <service-selector>
```

**Solution:** Ensure Service selector matches pod labels and pods are ready

### Issue: Insecure Traffic Warnings

**Problem:** Browser warns about insecure connection even with TLS enabled.

**Cause:** Using auto-generated certificate that browser doesn't trust.

**Solution:** For production, configure custom certificates or use cert-manager:
```yaml
tls:
  termination: edge
  certificate: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  key: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----
```

### Issue: Route Host Already Exists

**Problem:** Route creation fails with "host already claimed by another route."

**Cause:** Another route in another namespace is using the same hostname.

**Solution:** Use unique hostnames or enable namespace ownership validation:
```bash
# Check existing routes across namespaces
oc get routes --all-namespaces | grep <hostname>
```

### Issue: Backend Protocol Mismatch

**Problem:** Route serves garbled content or connection errors.

**Cause:** TLS termination mismatch (e.g., passthrough termination but backend expects HTTP).

**Solution:** Match termination type to backend protocol:
- Backend HTTP → `termination: edge`
- Backend HTTPS → `termination: passthrough` or `reencrypt`

## Alternatives to Routes

### Alternative 1: Kubernetes Ingress

OpenShift supports standard Kubernetes Ingress, but Routes are preferred.

**Pros:**
- Portable across Kubernetes distributions
- Standard API

**Cons:**
- Requires Ingress Controller configuration
- Less integrated with OpenShift features

**When to Use:** Multi-cloud deployments needing portability

### Alternative 2: NodePort Service

Expose service on all nodes at a static port.

**Pros:**
- Simple, no additional resources

**Cons:**
- Requires managing firewall rules
- Port conflicts across services
- No TLS termination
- Not production-grade

**When to Use:** Development only, quick testing

### Alternative 3: kubectl port-forward

Temporary port forwarding from local machine.

**Pros:**
- No cluster configuration needed
- Secure (uses kube-apiserver auth)

**Cons:**
- Not persistent
- Single-user
- Requires active kubectl session

**When to Use:** Development, debugging

## Testing Notes

### Verify Route Creation
```bash
oc get route -n <namespace>
oc describe route <route-name> -n <namespace>
```

### Get Route URL
```bash
ROUTE_URL=$(oc get route <route-name> -n <namespace> -o jsonpath='{.spec.host}')
echo "https://$ROUTE_URL"
```

### Test Route Connectivity
```bash
curl -k https://$(oc get route <route-name> -n <namespace> -o jsonpath='{.spec.host}')
```

### Check TLS Certificate
```bash
openssl s_client -connect $(oc get route <route-name> -n <namespace> -o jsonpath='{.spec.host}'):443 -servername $(oc get route <route-name> -n <namespace> -o jsonpath='{.spec.host}')
```

### Verify Backend Service
```bash
# Port-forward directly to service to test backend
oc port-forward svc/<service-name> 8080:3000 -n <namespace>
curl http://localhost:8080
```

## File Organization

```
deploy/helm/
└── <chart-name>/
    ├── templates/
    │   └── openshift.yaml         # Route resource
    └── values.yaml
        └── openshift:
              route:
                enabled: true|false
                host: ""
                tls:
                  termination: edge|passthrough|reencrypt
```

## Conversion from Upstream Patterns

### From kubectl port-forward

**Upstream documentation:**
```bash
kubectl port-forward svc/vss-service 3000:3000
# Access at http://localhost:3000
```

**OpenShift Route approach:**
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: vss-ui
spec:
  to:
    kind: Service
    name: vss-service
  port:
    targetPort: 3000
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

**Access:**
```bash
oc get route vss-ui -o jsonpath='{.spec.host}'
# Access at https://<route-host>
```

### From Kubernetes Ingress

**Upstream Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vss-ingress
spec:
  rules:
    - host: vss.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vss-service
                port:
                  number: 3000
```

**OpenShift Route equivalent:**
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: vss-ui
spec:
  host: vss.example.com
  to:
    kind: Service
    name: vss-service
  port:
    targetPort: 3000
  tls:
    termination: edge
```

## Best Practices

1. **Use Edge Termination**: Simplest TLS setup for most blueprints
2. **Enable HTTP→HTTPS Redirect**: `insecureEdgeTerminationPolicy: Redirect`
3. **Auto-Generate Hostnames**: Leave `host: ""` for development
4. **Reference Named Ports**: More maintainable than port numbers
5. **One Route Per UI**: Don't share routes across multiple services unless path-based routing is needed
6. **Document URL Retrieval**: Include `oc get route` command in deployment docs
7. **Test Before Documenting**: Verify route works before publishing docs
8. **Gate with Flags**: Use `openshift.route.enabled` for conditional creation

## Conversion Checklist

When adding OpenShift Routes to a blueprint:

- [ ] Identify services that need external access
- [ ] Create Route resource in `templates/openshift.yaml`
- [ ] Configure edge TLS termination with HTTP redirect
- [ ] Reference Service port by name (not number)
- [ ] Add `openshift.route.enabled` flag to values
- [ ] Gate Route creation behind `openshift.enabled`
- [ ] Document how to retrieve route URL (`oc get route`)
- [ ] Update README to replace port-forward instructions with Route access
- [ ] Test route connectivity from external network
- [ ] Verify TLS certificate is valid (or document self-signed warning)
- [ ] Add route URL to deployment verification section
