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
    approach: "A"
  - blueprint: "aiq"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/aiq"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-aiq"
    notes: "Overlay strategy with dedicated openshift.yaml template and nullable security contexts"
    approach: "B"
  - blueprint: "rag"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/rag"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-blueprint-enterprise-rag-pipeline"
    notes: "Overlay strategy with openshift.yaml template, demonstrates dynamic release naming for subchart service accounts in anyuid SCC RoleBinding"
    approach: "B"
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

---

## Approach B: Overlay Strategy (from aiq blueprint)

### When to Use

Use this approach when:
- You want **minimal changes to upstream templates**
- You're maintaining a fork and want easy upstream syncing
- You prefer **complete separation** between Kubernetes and OpenShift resources
- You want all OpenShift-specific resources in a **single dedicated file**

### Differences from Approach A

| Aspect | Approach A (Conditionals) | Approach B (Overlay) |
|--------|---------------------------|----------------------|
| Template modification | Conditionals scattered throughout | Only nullable security context |
| OpenShift resources | Inline with conditionals | Dedicated openshift.yaml file |
| Values configuration | Single values file | Overlay values-openshift.yaml |
| Security context | Conditionally rendered | Set to null in overlay |
| Upstream sync | Moderate conflict risk | Minimal conflict risk |

### Pattern Structure

#### 1. Dedicated OpenShift Template

Create `templates/openshift.yaml` that only renders when `openshift.enabled: true`:

```yaml
{{- $openshift := .Values.openshift | default dict }}
{{- if $openshift.enabled }}

# Route auto-conversion: checks route.enabled first, falls back to ingress.enabled
{{- range $appName, $appConfig := .Values.apps }}
{{- if $appConfig.enabled }}
{{- $appRoute := $appConfig.route | default dict }}
{{- $appIngress := $appConfig.ingress | default dict }}
{{- $routeEnabled := false }}
{{- if and (hasKey $appRoute "enabled") $appRoute.enabled }}
{{- $routeEnabled = true }}
{{- else }}
{{- if eq (kindOf $appIngress) "bool" }}
{{- $routeEnabled = $appIngress }}
{{- else if hasKey $appIngress "enabled" }}
{{- $routeEnabled = $appIngress.enabled }}
{{- end }}
{{- end }}

{{- if $routeEnabled }}
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ include "chart.appFullname" (list $ $appName) }}
spec:
  to:
    kind: Service
    name: {{ include "chart.appFullname" (list $ $appName) }}
  port:
    targetPort: {{ $appConfig.service.port }}
  {{- if $appRoute.tls }}
  tls:
    termination: {{ $appRoute.tls.termination | default "edge" }}
    insecureEdgeTerminationPolicy: {{ $appRoute.tls.insecureEdgeTerminationPolicy | default "Redirect" }}
  {{- end }}
{{- end }}
{{- end }}
{{- end }}

---
# Grant anyuid SCC to all app service accounts
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ $.Values.project.name }}-anyuid-scc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:anyuid
subjects:
  - kind: ServiceAccount
    name: default
{{- range $appName, $appConfig := .Values.apps }}
{{- if $appConfig.enabled }}
  - kind: ServiceAccount
    name: {{ include "chart.appFullname" (list $ $appName) }}
{{- end }}
{{- end }}

{{- $ngcSecret := $openshift.ngcSecret | default dict }}
{{- if $ngcSecret.password }}
---
# NGC image pull secret from values
apiVersion: v1
kind: Secret
metadata:
  name: ngc-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ printf `{"auths":{"nvcr.io":{"auth":"%s"}}}` (printf "$oauthtoken:%s" $ngcSecret.password | b64enc) | b64enc }}
{{- end }}

{{- $apiKeys := $openshift.apiKeys | default dict }}
---
# Application credentials secret from values
apiVersion: v1
kind: Secret
metadata:
  name: {{ $.Values.project.name }}-credentials
type: Opaque
stringData:
  NVIDIA_API_KEY: {{ $apiKeys.nvidiaApiKey | default "" | quote }}
  TAVILY_API_KEY: {{ $apiKeys.tavilyApiKey | default "" | quote }}
  DB_USER_NAME: {{ $apiKeys.dbUserName | default "app_user" | quote }}
  DB_USER_PASSWORD: {{ $apiKeys.dbUserPassword | default "changeme" | quote }}

{{- end }}
```

#### 2. Nullable Security Context Pattern

In `templates/deployment.yaml`, use a double-check pattern to allow null values:

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      # Pod-level security context
      {{- if hasKey $appConfig "podSecurityContext" }}
      {{- if $appConfig.podSecurityContext }}
      securityContext:
        {{- toYaml $appConfig.podSecurityContext | nindent 8 }}
      {{- end }}
      {{- else }}
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      {{- end }}
      
      containers:
        - name: {{ $appName }}
          # Container-level security context
          {{- if hasKey $appConfig "securityContext" }}
          {{- if $appConfig.securityContext }}
          securityContext:
            {{- toYaml $appConfig.securityContext | nindent 12 }}
          {{- end }}
          {{- else }}
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: false
          {{- end }}
```

**Why double-check?**
- First `if hasKey` - checks if key exists in values
- Second `if <value>` - checks if value is truthy (non-null)
- This allows `podSecurityContext: null` to suppress the security context block completely

#### 3. Overlay Values File

Create `values-openshift.yaml` that's applied with `-f` flag:

```yaml
# OpenShift value overrides
#
# Usage:
#   helm install app ./chart \
#     -f values-openshift.yaml \
#     --set openshift.ngcSecret.password="$NGC_API_KEY" \
#     --set openshift.apiKeys.nvidiaApiKey="$NVIDIA_API_KEY"

openshift:
  enabled: true
  
  ngcSecret:
    password: ""  # Pass via --set to avoid storing in files
  
  apiKeys:
    nvidiaApiKey: ""
    tavilyApiKey: ""
    dbUserName: "app_user"
    dbUserPassword: "changeme"

apps:
  backend:
    imagePullSecrets:
      - name: ngc-secret
    ingress:
      enabled: false
    route:
      enabled: true
      tls:
        termination: edge
        insecureEdgeTerminationPolicy: Redirect
    podSecurityContext: null  # Let OpenShift SCC manage it
    securityContext: null
  
  frontend:
    imagePullSecrets:
      - name: ngc-secret
    ingress:
      enabled: false
    route:
      enabled: true
      tls:
        termination: edge
    podSecurityContext: null
    securityContext: null
  
  database:
    podSecurityContext: null
    securityContext: null
```

#### 4. Base Values Configuration

In `values.yaml`, add minimal OpenShift section:

```yaml
openshift:
  enabled: false  # Set to true to enable OpenShift-compatible deployment
  ngcSecret:
    password: ""
  apiKeys:
    nvidiaApiKey: ""
    tavilyApiKey: ""
    dbUserName: "app_user"
    dbUserPassword: "changeme"
```

### Usage

**Deploy to OpenShift:**
```bash
helm install app ./chart \
  -f values-openshift.yaml \
  --set openshift.ngcSecret.password="$NGC_API_KEY" \
  --set openshift.apiKeys.nvidiaApiKey="$NVIDIA_API_KEY" \
  --set openshift.apiKeys.tavilyApiKey="$TAVILY_API_KEY" \
  --namespace my-namespace
```

**Deploy to Standard Kubernetes:**
```bash
helm install app ./chart \
  --namespace my-namespace
```

### Key Patterns

#### 1. Route Auto-Conversion Logic

The route creation checks `route.enabled` first, then falls back to `ingress.enabled` for backward compatibility:

```yaml
{{- $routeEnabled := false }}
{{- if and (hasKey $appRoute "enabled") $appRoute.enabled }}
{{- $routeEnabled = true }}
{{- else }}
{{- if eq (kindOf $appIngress) "bool" }}
{{- $routeEnabled = $appIngress }}
{{- else if hasKey $appIngress "enabled" }}
{{- $routeEnabled = $appIngress.enabled }}
{{- end }}
{{- end }}
```

This allows existing Kubernetes configurations with `ingress.enabled: true` to automatically get Routes on OpenShift.

#### 2. Dynamic SCC Binding

The anyuid RoleBinding dynamically includes all enabled app service accounts:

```yaml
subjects:
  - kind: ServiceAccount
    name: default
{{- range $appName, $appConfig := .Values.apps }}
{{- if $appConfig.enabled }}
  - kind: ServiceAccount
    name: {{ include "chart.appFullname" (list $ $appName) }}
{{- end }}
{{- end }}
```

No need to manually list service accounts - they're auto-discovered from `.Values.apps`.

#### 3. Declarative Secret Creation

Secrets are created from values passed via `--set` flags:

```yaml
{{- if $ngcSecret.password }}
---
apiVersion: v1
kind: Secret
metadata:
  name: ngc-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ printf `{"auths":{"nvcr.io":{"auth":"%s"}}}` (printf "$oauthtoken:%s" $ngcSecret.password | b64enc) | b64enc }}
{{- end }}
```

This eliminates manual `oc create secret` commands - single `helm install` does everything.

### Benefits

1. **Minimal Upstream Impact**: Only one change to core templates (nullable security context)
2. **Easy Fork Maintenance**: Upstream changes rarely conflict with OpenShift overlay
3. **Clear Separation**: All OpenShift resources in one file, easy to review
4. **Single Command Deploy**: No manual secret creation steps
5. **Auto-Discovery**: Service accounts and routes auto-configured from enabled apps

### Gotchas

1. **Security context must be `null`, not `{}`**: Empty dict would still render the key
2. **Overlay file applied with `-f` flag**: Not merged into a subkey
3. **Double-check pattern required**: Both `hasKey` and value check needed for nullable fields
4. **Use dynamic release names for subchart service accounts**: Avoid hardcoding service account names in RoleBindings/SCCs to allow multiple releases in the same cluster

#### Dynamic Release Naming for Service Accounts

**Problem:** Hardcoded service account names in RoleBindings/SCCs only work for one release name, preventing multiple instances of the same blueprint from coexisting in a cluster.

**Bad Example (hardcoded):**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "nvidia-blueprint-rag.fullname" . }}-anyuid-scc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:anyuid
subjects:
  - kind: ServiceAccount
    name: rag-nv-ingest  # ❌ Only works for release named "rag"
  - kind: ServiceAccount
    name: rag-minio      # ❌ Only works for release named "rag"
  - kind: ServiceAccount
    name: rag-redis-master  # ❌ Only works for release named "rag"
```

**Good Example (dynamic naming):**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "nvidia-blueprint-rag.fullname" . }}-anyuid-scc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:anyuid
subjects:
  - kind: ServiceAccount
    name: {{ .Release.Name }}-nv-ingest  # ✅ Works for any release name
  - kind: ServiceAccount
    name: {{ .Release.Name }}-minio      # ✅ Works for any release name
  - kind: ServiceAccount
    name: {{ .Release.Name }}-redis-master  # ✅ Works for any release name
```

**Why This Matters:**
- Allows `helm install rag1 ...` and `helm install rag2 ...` in the same namespace
- Enables testing different configurations side-by-side
- Matches Helm's subchart naming convention (subcharts automatically prefix resources with release name)

**Exception for NIM Operator:**
The `nim-cache-sa` ServiceAccount is created by the NIM Operator with a fixed name (not by the Helm chart), so it doesn't use the release name prefix:

```yaml
subjects:
  - kind: ServiceAccount
    name: nim-cache-sa  # Fixed name - created by NIM Operator, not Helm
```

**How to Identify Service Account Names:**

For Bitnami subcharts (Redis, MinIO, PostgreSQL, MongoDB):
```yaml
{{ .Release.Name }}-<subchart-name>-<component>
```

For NVIDIA subcharts (nv-ingest):
```yaml
{{ .Release.Name }}-<subchart-name>
```

For custom subcharts, check the subchart's `templates/serviceaccount.yaml`:
```yaml
name: {{ include "subchart.serviceAccountName" . }}
```

**Real-World Fix from RAG Blueprint:**
Commit ca9844e fixed this issue with the message: "fix: use dynamic release name for OpenShift SCC service accounts"

**Before:**
- Could only install with `helm install rag ...`
- Installing `helm install rag-dev ...` would fail (SCC not granted to correct service accounts)

**After:**
- Works with any release name: `rag`, `rag-dev`, `rag-prod`, etc.
- Multiple instances can coexist in same namespace

## Choosing Between Approaches

### Use Approach A (Conditional Templates) when:
- You control the upstream repository
- You want a single values file
- You prefer explicit conditionals over overlay files
- Your team is comfortable with scattered conditionals

### Use Approach B (Overlay Strategy) when:
- You're maintaining a fork of an upstream chart
- You want minimal merge conflicts with upstream
- You prefer complete separation of concerns
- You want all OpenShift resources in one place for easy review
- You're planning to submit OpenShift support as a PR upstream (cleaner diff)

## Related Patterns

- [[security-contexts-scc]] - Security context details
- [[networking-routes-ingress]] - Route configuration patterns
- [[mongodb-on-rhoai]] - Example component using Approach A
- [[redis-on-rhoai]] - Example component using Approach A
