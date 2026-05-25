---
name: helm-openshift-conditionals
description: Helm chart pattern with openshift.enabled flag and template helper functions for dual-mode deployment
metadata:
  type: deployment-pattern
deployment_types: [helm]
resource_types: [security-context, networking, storage]
source_examples:
  - blueprint: "data-flywheel"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/data-flywheel"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-data-flywheel"
    notes: "Complete example of Helm chart with openshift.enabled conditional support"
---

# Helm Charts with OpenShift Conditional Support

## Overview

This pattern demonstrates how to create Helm charts that can deploy to both standard Kubernetes and OpenShift using a single `openshift.enabled` flag. The chart conditionally applies OpenShift-specific configurations while remaining compatible with standard Kubernetes.

## Pattern Structure

### 1. Values Configuration

Add an `openshift` section to `values.yaml`:

```yaml
openshift:
  enabled: false  # Set to true to enable OpenShift-compatible deployment
  
  # Security Context configuration for OpenShift restricted SCC
  securityContext:
    # Pod-level security context
    pod:
      runAsNonRoot: true
      # Note: fsGroup is automatically assigned by OpenShift from namespace UID/GID range
      # Do not set fsGroup explicitly as it conflicts with restricted-v2 SCC
      seccompProfile:
        type: RuntimeDefault
    
    # Container-level security context
    container:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      capabilities:
        drop:
          - ALL
  
  # Route configuration for external access
  routes:
    enabled: true
    api:
      enabled: true
      host: ""  # Auto-generated if empty
      tls:
        enabled: true
        termination: edge
        insecureEdgeTerminationPolicy: Redirect
  
  # Storage class override for OpenShift
  storageClass: "gp3-csi"
  
  # Init container image for OpenShift compatibility
  initContainer:
    image: "registry.access.redhat.com/ubi8/ubi-minimal:latest"
```

### 2. Helper Functions

Create reusable helper functions in `templates/_helpers.tpl`:

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

### 3. Deployment Template Pattern

Apply conditionals in deployment templates:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.myService.fullnameOverride }}-deployment
spec:
  replicas: {{ .Values.myService.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.myService.fullnameOverride }}-deployment
  template:
    metadata:
      labels:
        app: {{ .Values.myService.fullnameOverride }}-deployment
    spec:
      automountServiceAccountToken: false
      
      # Pod-level security context (OpenShift only)
      {{- if .Values.openshift.enabled }}
      securityContext:
        {{- include "data-flywheel.podSecurityContext" . | nindent 8 }}
      {{- end }}
      
      # Conditional init containers
      initContainers:
        {{- if .Values.openshift.enabled }}
        - name: create-data-folder
          image: {{ .Values.openshift.initContainer.image }}
          command: ['sh', '-c', 'mkdir -p /data']
          volumeMounts:
            - name: data-volume
              mountPath: /data
          securityContext:
            {{- include "data-flywheel.containerSecurityContext" . | nindent 12 }}
        {{- else }}
        - name: create-data-folder
          image: busybox:1.28
          command: ['sh', '-c', 'mkdir -p /data && chmod 755 /data']
          volumeMounts:
            - name: data-volume
              mountPath: /data
        {{- end }}
      
      containers:
        - name: my-container
          image: "{{ .Values.myService.image.repository }}:{{ .Values.myService.image.tag }}"
          imagePullPolicy: Always
          
          # Container-level security context (OpenShift only)
          {{- if .Values.openshift.enabled }}
          securityContext:
            {{- include "data-flywheel.containerSecurityContext" . | nindent 12 }}
          {{- end }}
          
          # Conditional environment variables for restricted environments
          env:
            - name: MY_ENV_VAR
              value: "value"
            {{- if .Values.openshift.enabled }}
            - name: HOME
              value: "/tmp"
            - name: UV_CACHE_DIR
              value: "/tmp/.uv-cache"
            - name: UV_PROJECT_ENVIRONMENT
              value: "/tmp/.venv"
            {{- end }}
          
          volumeMounts:
            - name: data-volume
              mountPath: /data
      
      volumes:
        - name: data-volume
          emptyDir: {}
```

### 4. Service Template Pattern

Services switch type based on OpenShift mode:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.myService.fullnameOverride }}-service
spec:
  selector:
    app: {{ .Values.myService.fullnameOverride }}-deployment
  type: {{ include "data-flywheel.serviceType" . }}
  ports:
    - port: {{ .Values.myService.service.port }}
```

### 5. Route Template Pattern

Create separate Route resources for OpenShift:

**File**: `templates/my-service-route.yaml`
```yaml
{{- if and .Values.openshift.enabled .Values.openshift.routes.enabled .Values.openshift.routes.myService.enabled .Values.myService.enabled }}
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ .Values.myService.fullnameOverride }}-route
  labels:
    app: {{ .Values.myService.fullnameOverride }}-deployment
spec:
  {{- if .Values.openshift.routes.myService.host }}
  host: {{ .Values.openshift.routes.myService.host }}
  {{- end }}
  to:
    kind: Service
    name: {{ .Values.myService.fullnameOverride }}-service
    weight: 100
  port:
    targetPort: {{ .Values.myService.service.port }}
  {{- if .Values.openshift.routes.myService.tls.enabled }}
  tls:
    termination: {{ .Values.openshift.routes.myService.tls.termination }}
    insecureEdgeTerminationPolicy: {{ .Values.openshift.routes.myService.tls.insecureEdgeTerminationPolicy }}
  {{- end }}
  wildcardPolicy: None
{{- end }}
```

## Usage

### Deploy to OpenShift

```bash
helm install my-app ./chart \
  --set openshift.enabled=true \
  --set namespace=my-namespace
```

### Deploy to Standard Kubernetes

```bash
helm install my-app ./chart \
  --set openshift.enabled=false \
  --set namespace=my-namespace
```

## Key Patterns

### 1. Security Context Abstraction

- Use helper functions to centralize security context logic
- Apply pod and container security contexts only when `openshift.enabled=true`
- Don't set `fsGroup` explicitly; let OpenShift assign it

### 2. Init Container Image Switching

- **OpenShift**: Use Red Hat UBI minimal image
- **Standard K8s**: Use busybox
- **Why**: Restricted SCC prevents `chmod` in OpenShift, so different commands are needed

### 3. Service Type Switching

- **OpenShift**: ClusterIP (external access via Routes)
- **Standard K8s**: NodePort (external access without ingress)

### 4. Environment Variable Adjustments

For Python/UV-based applications in restricted environments:
```yaml
{{- if .Values.openshift.enabled }}
- name: HOME
  value: "/tmp"
- name: UV_CACHE_DIR
  value: "/tmp/.uv-cache"
- name: UV_PROJECT_ENVIRONMENT
  value: "/tmp/.venv"
{{- end }}
```

**Why?** Restricted SCC prevents writing to default home directory; redirect to `/tmp`.

### 5. Conditional Route Creation

- Only create Routes when all conditions are met:
  - `openshift.enabled=true`
  - `openshift.routes.enabled=true`
  - `openshift.routes.<serviceName>.enabled=true`
  - The service itself is enabled

## Benefits

1. **Single Chart**: One Helm chart works for both Kubernetes and OpenShift
2. **Easy Switching**: Toggle with a single flag
3. **No Code Duplication**: Shared logic via helper functions
4. **Maintainability**: Changes to security contexts apply everywhere via helpers
5. **Gradual Migration**: Can test OpenShift mode before fully migrating

## Related Patterns

- [[security-contexts-scc]] - Security context details
- [[networking-routes-ingress]] - Route configuration patterns
- [[mongodb-on-rhoai]] - Example component using this pattern
- [[redis-on-rhoai]] - Example component using this pattern
