---
type: deployment-pattern
deployment_types: [rhoai-notebook]
components: [isaac-lab, jupyter]
resource_types: [gpu, storage]
source_examples:
  - blueprint: "synthetic-manipulation-motion-generation"
    source_repo: "https://github.com/NVIDIA-Omniverse-blueprints/synthetic-manipulation-motion-generation"
    fork_repo: "https://github.com/rh-ai-quickstart/synthetic-manipulation-motion-generation"
    notes: "First RHOAI notebook pattern with Xvfb, GPU, PVC initialization"
---

# RHOAI Notebook Deployment Pattern

## Overview

This pattern is for blueprints that are primarily notebook-based workflows (Jupyter notebooks) and should be deployed as custom RHOAI notebook images, not as Helm charts or standalone services.

**Use this pattern when:**
- The blueprint's primary interface is a Jupyter notebook
- Users interact with the blueprint through a notebook UI
- The blueprint includes data science/ML workflows
- GPU access is required for interactive work

**Don't use this pattern when:**
- The blueprint is a service/API deployment (use Helm pattern instead)
- No notebook interface exists
- The blueprint is batch-only (no interactive component)

## File Structure

```
blueprint/
├── Dockerfile                          # Custom notebook image
├── deploy/
│   ├── deploy-isaac-lab-image.sh      # Deployment script
│   └── imagestream.yaml                # RHOAI ImageStream definition
├── launch.sh                           # Entrypoint with OPENSHIFT_MODE logic
├── notebook/
│   └── *.ipynb                         # Jupyter notebooks
└── docker-compose.yml                  # Original deployment (preserved)
```

## Conversion Steps

### 1. Create Custom Dockerfile

**Purpose:** Build a notebook image that extends the NVIDIA base image and adds RHOAI compatibility.

**Template:**
```dockerfile
# Start from NVIDIA base image
FROM nvcr.io/nvidia/<blueprint-base-image>:<version>

# Install any headless rendering dependencies (if needed)
RUN apt-get update && \
    apt-get install -y xvfb x11-utils && \
    rm -rf /var/lib/apt/lists/*

# Set environment variables for OpenShift mode
ENV OPENSHIFT_MODE=true \
    DISPLAY=:99 \
    ACCEPT_EULA=Y \
    HOME=/opt/app-root/src

# Copy modified launch script
COPY launch.sh /path/to/launch.sh
RUN chmod +x /path/to/launch.sh

# Install Jupyter Lab at build time
RUN python3 -m pip install jupyter

# Copy all notebook files (replaces docker-compose volume mounts)
COPY notebook/*.ipynb /path/to/notebooks/
COPY notebook/*.py /path/to/notebooks/

# Copy any input data or samples
COPY samples/*.hdf5 /path/to/data/

# Set permissions for OpenShift compatibility
# OpenShift runs as random UID but always group 0
RUN chgrp -R 0 /opt/app-root/src && \
    chmod -R g=u /opt/app-root/src

# Make any application-specific directories writable for group 0
# (identify these by running the app and checking for permission errors)
RUN chmod -R g+w /path/to/writable/dirs

# Set working directory
WORKDIR /path/to/notebooks

# Entrypoint
ENTRYPOINT ["/path/to/launch.sh"]
```

**Key points:**
- Set `HOME=/opt/app-root/src` (RHOAI standard)
- Set `OPENSHIFT_MODE=true` to trigger RHOAI-specific behavior
- Install Jupyter Lab in the image
- Copy files to non-PVC locations (they'll be copied to PVC at runtime)
- Set group 0 permissions (`chgrp -R 0`, `chmod -R g=u`)

### 2. Create ImageStream YAML

**Purpose:** Register the notebook image in RHOAI UI so users can select it when creating workbenches.

**Template (`deploy/imagestream.yaml`):**
```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: <blueprint-name>
  namespace: redhat-ods-applications
  labels:
    opendatahub.io/dashboard: "true"
    opendatahub.io/notebook-image: "true"
    opendatahub.io/component: "true"
    app.kubernetes.io/part-of: "workbenches"
    component.opendatahub.io/name: "notebooks"
  annotations:
    opendatahub.io/notebook-image-name: "<Human-Readable Name>"
    opendatahub.io/notebook-image-desc: "<Description> (GPU required)"
spec:
  lookupPolicy:
    local: false
  tags:
  - name: "1.0"
    annotations:
      opendatahub.io/notebook-software: '[{"name":"Python","version":"3.10"},{"name":"CUDA","version":"12.1"}]'
      opendatahub.io/notebook-python-dependencies: '[{"name":"<package>","version":"<version>"}]'
    from:
      kind: ImageStreamTag
      namespace: ${NAMESPACE}
      name: <blueprint-name>:1.0
    referencePolicy:
      type: Source
```

**Key fields:**
- `metadata.namespace: redhat-ods-applications`: Required for RHOAI UI integration
- `metadata.labels`: All required for notebook to appear in UI
- `annotations.opendatahub.io/notebook-image-name`: Display name in UI
- `annotations.opendatahub.io/notebook-image-desc`: Description (note GPU requirement if needed)
- `annotations.opendatahub.io/notebook-software`: Software versions (shown in UI)
- `annotations.opendatahub.io/notebook-python-dependencies`: Key packages (shown in UI)

### 3. Create Deployment Script

**Purpose:** Automate the build and deployment process.

**Template (`deploy/deploy-<name>-image.sh`):**
```bash
#!/bin/bash
set -e

# Default namespace to nvidia-omniverse-blueprint if not set
NAMESPACE=${NAMESPACE:-nvidia-omniverse-blueprint}

echo "=== <Blueprint Name> RHOAI Deployment ==="
echo "Using namespace: $NAMESPACE"

# Create BuildConfig
echo "Creating BuildConfig..."
oc new-build --binary --strategy=docker --name <blueprint-name> -n "$NAMESPACE"

# Build image
echo "Building image..."
oc start-build <blueprint-name> --from-dir=. --follow -n "$NAMESPACE"

# Create RHOAI ImageStream
echo "Creating RHOAI ImageStream..."
envsubst < deploy/imagestream.yaml | oc apply -f -

# Tag :latest as :1.0
echo "Tagging image as 1.0..."
oc tag "$NAMESPACE/<blueprint-name>:latest" "$NAMESPACE/<blueprint-name>:1.0"

echo ""
echo "✓ Deployment complete!"
echo "✓ Image ready in RHOAI"
echo ""
echo "Next: Create workbench in RHOAI UI and select '<Blueprint Name>' image"
```

**Usage:**
```bash
export NAMESPACE=my-namespace
./deploy/deploy-<name>-image.sh
```

### 4. Modify Launch Script

**Purpose:** Add RHOAI-specific initialization logic while preserving original docker-compose behavior.

**Template (`launch.sh`):**
```bash
#!/bin/bash
set -e

# OpenShift Mode - handle and exit
if [ "$OPENSHIFT_MODE" = "true" ]; then
    echo "Running in OpenShift mode"

    # Copy notebook files to PVC on first startup
    if [ ! -f /opt/app-root/src/<key-notebook>.ipynb ]; then
        echo "First startup - copying notebook files to PVC..."
        cp /path/to/notebooks/*.ipynb /opt/app-root/src/
        cp /path/to/notebooks/*.py /opt/app-root/src/
        echo "Notebook files ready"
    fi

    # Create user data directories on PVC (must be at runtime, not build time)
    echo "Creating user data directories on PVC..."
    mkdir -p /opt/app-root/src/datasets \
             /opt/app-root/src/output

    # Copy input data to PVC on first startup (if applicable)
    if [ ! -f /opt/app-root/src/datasets/<input-file> ]; then
        echo "Copying input dataset to PVC..."
        cp /path/to/data/<input-file> /opt/app-root/src/datasets/
        echo "Input dataset ready"
    fi

    # If headless rendering is needed (Xvfb):
    echo "Starting Xvfb virtual display..."
    Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset &
    XVFB_PID=$!
    sleep 2
    if ! ps -p $XVFB_PID > /dev/null; then
        echo "ERROR: Xvfb failed to start"
        exit 1
    fi
    export DISPLAY=:99
    cleanup() {
        echo "Stopping Xvfb..."
        kill $XVFB_PID 2>/dev/null || true
    }
    trap cleanup EXIT

    # Start Jupyter Lab (non-root mode for OpenShift)
    # RHOAI injects NOTEBOOK_ARGS with correct base_url, token, password, port
    <python-interpreter> -m jupyter lab \
        --ip=0.0.0.0 \
        --no-browser \
        $NOTEBOOK_ARGS

    exit 0
fi

# Running in docker-compose mode
# ... original launch logic ...
```

**Pattern explanation:**
- Check `OPENSHIFT_MODE` environment variable at start
- If true, execute RHOAI-specific logic and exit
- If false, fall through to original docker-compose logic
- This preserves both deployment modes in a single script

**Critical details:**
- Copy files to PVC only on first startup (check for existence)
- Create directories at runtime, not build time (PVC mount overwrites build-time content)
- Use `$NOTEBOOK_ARGS` (injected by RHOAI) when launching Jupyter Lab
- Do NOT use `--allow-root` flag (OpenShift runs as non-root)

### 5. Update Notebooks for OPENSHIFT_MODE

**Purpose:** Notebooks may need to adapt behavior based on deployment mode.

**Pattern:**
```python
import os

openshift_mode = os.getenv("OPENSHIFT_MODE", "false").lower() == "true"

if openshift_mode:
    # RHOAI-specific configuration
    # - Use headless mode
    # - Use managed services (NIM API)
    # - Adjust paths for PVC
    pass
else:
    # Original configuration
    # - Interactive mode
    # - Self-hosted services
    pass
```

**Example from Isaac Lab:**
```python
# Use headless mode only in OpenShift (Xvfb) environment
openshift_mode = os.getenv("OPENSHIFT_MODE", "false").lower() == "true"
headless_args = ["--headless"] if openshift_mode else []
args_cli = parser.parse_args(headless_args)

# Conditionally disable features that don't work in headless mode
config = {}
if not openshift_mode:
    config["enable"] = "omni.kit.renderer.capture"
```

### 6. Update README

Add RHOAI deployment instructions to README.md:

```markdown
## Deployment on Red Hat OpenShift AI

### Prerequisites
- Red Hat OpenShift AI installed
- GPU-enabled OpenShift cluster
- Namespace with GPU quotas

### Deploy Custom Notebook Image

```bash
export NAMESPACE=<your-namespace>
./deploy/deploy-<name>-image.sh
```

### Create Workbench

1. Log in to RHOAI UI
2. Create a new workbench
3. Select "<Blueprint Name>" notebook image
4. Choose GPU size (1 GPU minimum)
5. Create workbench
6. Wait for workbench to start
7. Open `<main-notebook>.ipynb` and run cells

### Verify

```bash
# Check ImageStream
oc get imagestream <blueprint-name> -n redhat-ods-applications

# Check build
oc get builds -n <namespace>
```
```

## OPENSHIFT_MODE Conditional Pattern

### Environment Variable Detection

All RHOAI-specific logic is gated behind `OPENSHIFT_MODE=true` check:

**Shell scripts:**
```bash
if [ "$OPENSHIFT_MODE" = "true" ]; then
    # RHOAI logic
    exit 0
fi
# Original logic
```

**Python:**
```python
import os
openshift_mode = os.getenv("OPENSHIFT_MODE", "false").lower() == "true"
if openshift_mode:
    # RHOAI logic
else:
    # Original logic
```

**Why this pattern:**
- Preserves original deployment mode (docker-compose)
- Single codebase supports both modes
- Clear separation of concerns
- Easy to test both modes

## PVC Initialization Pattern

### Problem

RHOAI mounts a PVC at `/opt/app-root/src` (HOME directory). This mount **replaces any content created at build time**.

### Solution

1. **In Dockerfile:** Copy files to a non-PVC location (e.g., `/workspace/`, `/app/`)
2. **In launch.sh:** On first startup, copy files from image to PVC

**Why this works:**
- Files baked into image remain accessible at non-PVC location
- Runtime copy to PVC makes files visible to Jupyter user
- Check for existence prevents re-copying on subsequent startups
- User modifications persist across pod restarts

**Template:**
```bash
if [ "$OPENSHIFT_MODE" = "true" ]; then
    # Copy files to PVC on first startup
    if [ ! -f /opt/app-root/src/<marker-file> ]; then
        echo "First startup - copying files to PVC..."
        cp /source/path/* /opt/app-root/src/
        echo "Files ready"
    fi
fi
```

## Security Context for Random UID

### Problem

OpenShift runs containers as a random UID (not root, not the UID in the Dockerfile) but always with group 0.

### Solution

Make all writable directories/files accessible to group 0:

```dockerfile
# Set group ownership to 0 (root group)
RUN chgrp -R 0 /path/to/writable/dir && \
    chmod -R g=u /path/to/writable/dir

# For specific directories, just add group write
RUN chmod -R g+w /path/to/writable/dir
```

**Common directories that need this:**
- `/opt/app-root/src` (HOME)
- Application cache directories
- Application log directories
- Application data directories

**How to identify them:**
- Run the image on OpenShift
- Check for permission errors in logs
- Add group write permissions for those directories
- Rebuild and test

## Jupyter Lab Integration

### NOTEBOOK_ARGS Environment Variable

RHOAI injects `NOTEBOOK_ARGS` with:
- `--NotebookApp.base_url='/notebook/<namespace>/<workbench-name>'`
- `--NotebookApp.token='...'`
- `--NotebookApp.password='...'`
- `--port=8888`

**Usage in launch.sh:**
```bash
<python-interpreter> -m jupyter lab \
    --ip=0.0.0.0 \
    --no-browser \
    $NOTEBOOK_ARGS
```

**Important:**
- Use the application's Python interpreter to preserve environment
- Do NOT use `--allow-root` (runs as non-root random UID)
- Do NOT set custom token/password (RHOAI manages this)
- Pass `$NOTEBOOK_ARGS` without quotes (it contains multiple flags)

## Known Issues and Gotchas

### Issue: Build fails with "permission denied" on BuildConfig

**Solution:** Ensure current directory is the repository root when running deploy script.

```bash
cd /path/to/blueprint-root
./deploy/deploy-<name>-image.sh
```

### Issue: Notebook image doesn't appear in RHOAI UI

**Possible causes:**
1. ImageStream not in `redhat-ods-applications` namespace
2. Missing required labels (check `metadata.labels`)
3. Missing required annotations (check `metadata.annotations`)

**Solution:** Verify ImageStream matches template exactly.

```bash
oc get imagestream -n redhat-ods-applications
oc describe imagestream <name> -n redhat-ods-applications
```

### Issue: Jupyter Lab fails to start with "could not bind to port"

**Cause:** Port 8888 is hardcoded, but RHOAI uses dynamic ports.

**Solution:** Do NOT specify `--port` in launch.sh. Let RHOAI inject it via `$NOTEBOOK_ARGS`.

### Issue: User modifications to notebooks lost on pod restart

**Cause:** Files are in ephemeral storage, not PVC.

**Solution:** Ensure notebooks are in `/opt/app-root/src` (which is the PVC mount point).

### Issue: Application writes fail with "permission denied"

**Cause:** Application directories are not writable by group 0.

**Solution:** Add `chmod -R g+w /path/to/dir` in Dockerfile for all directories the application writes to.

## Testing Checklist

- [ ] ImageStream appears in RHOAI UI
- [ ] Workbench creation succeeds
- [ ] Jupyter Lab starts without errors
- [ ] Notebooks load successfully
- [ ] First startup copies files to PVC
- [ ] Subsequent startups skip file copy
- [ ] User modifications persist across pod restarts
- [ ] GPU is accessible (if required)
- [ ] Application runs without permission errors

## Related Patterns

- [components/isaac-lab-on-rhoai.md](../components/isaac-lab-on-rhoai.md) - Example implementation
- [resource-patterns/gpu-allocation-openshift.md](../resource-patterns/gpu-allocation-openshift.md) - GPU configuration
- [resource-patterns/security-contexts-scc.md](../resource-patterns/security-contexts-scc.md) - Security contexts
