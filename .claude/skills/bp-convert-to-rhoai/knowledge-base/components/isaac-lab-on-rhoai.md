---
type: component
components: [isaac-lab, isaac-sim]
deployment_types: [rhoai-notebook]
resource_types: [gpu, storage]
architecture: [notebook-based]
source_examples:
  - blueprint: "synthetic-manipulation-motion-generation"
    source_repo: "https://github.com/NVIDIA-Omniverse-blueprints/synthetic-manipulation-motion-generation"
    fork_repo: "https://github.com/rh-ai-quickstart/synthetic-manipulation-motion-generation"
    notes: "Demonstrates RHOAI notebook pattern with headless rendering (Xvfb), GPU, PVC initialization"
    approach: "A"
---

# Isaac Lab / Isaac Sim on RHOAI

## Overview

Isaac Lab is NVIDIA's robotics simulation framework built on top of Isaac Sim. Running it on RHOAI requires:
- Headless rendering using Xvfb (no physical display)
- GPU allocation
- Persistent volume for user data and notebooks
- Runtime initialization of directories on PVC
- Custom launcher script that detects OPENSHIFT_MODE

This pattern applies to any GPU-accelerated simulation or rendering workload that expects a display.

## Conversion Pattern

### OPENSHIFT_MODE Conditional Support

The blueprint uses `OPENSHIFT_MODE=true` environment variable to trigger RHOAI-specific behavior across multiple files:

**In Dockerfile:**
```dockerfile
ENV OPENSHIFT_MODE=true \
    DISPLAY=:99 \
    ACCEPT_EULA=Y \
    HOME=/opt/app-root/src
```

**In launch.sh:**
```bash
if [ "$OPENSHIFT_MODE" = "true" ]; then
    echo "Running in OpenShift mode (headless with Xvfb)"
    # ... RHOAI-specific initialization
    exit 0
fi

# Running in docker-compose mode
# ... original logic
```

**In Python notebooks:**
```python
import os
openshift_mode = os.getenv("OPENSHIFT_MODE", "false").lower() == "true"

# Use headless mode only in OpenShift
headless_args = ["--headless"] if openshift_mode else []
args_cli = parser.parse_args(headless_args)
```

### Deployment Type: RHOAI Notebook

This component is deployed as a custom RHOAI notebook image, not as a Helm chart or standalone deployment.

**Key steps:**
1. Build custom Dockerfile that extends NVIDIA base image
2. Deploy using OpenShift BuildConfig
3. Create RHOAI ImageStream to register notebook in UI
4. Users create workbenches in RHOAI UI and select this image

See [deployment-types/rhoai-notebook-pattern.md](../deployment-types/rhoai-notebook-pattern.md) for details.

### Headless Rendering with Xvfb

Isaac Sim requires a display for rendering. On RHOAI (no physical display), use Xvfb virtual framebuffer.

**In Dockerfile, install Xvfb:**
```dockerfile
RUN apt-get update && \
    apt-get install -y xvfb x11-utils && \
    rm -rf /var/lib/apt/lists/*

ENV DISPLAY=:99
```

**In launch.sh, start Xvfb before launching Isaac Sim:**
```bash
if [ "$OPENSHIFT_MODE" = "true" ]; then
    # Start Xvfb for headless rendering
    echo "Starting Xvfb virtual display..."
    Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset &
    XVFB_PID=$!

    # Wait for Xvfb to start
    sleep 2

    # Verify Xvfb is running
    if ! ps -p $XVFB_PID > /dev/null; then
        echo "ERROR: Xvfb failed to start"
        exit 1
    fi

    # Export DISPLAY for Isaac Sim
    export DISPLAY=:99

    # Cleanup function for Xvfb
    cleanup() {
        echo "Stopping Xvfb..."
        kill $XVFB_PID 2>/dev/null || true
    }
    trap cleanup EXIT

    # Launch application...
fi
```

### Security Context Requirements

OpenShift runs containers as random UID but always with group 0 (root group).

**In Dockerfile:**
```dockerfile
# Set permissions for OpenShift compatibility
# OpenShift runs as random UID but always group 0
RUN chgrp -R 0 /opt/app-root/src && \
    chmod -R g=u /opt/app-root/src && \
    chmod -R 775 /workspace/isaaclab

# Make Isaac Sim directories writable for OpenShift (runs as random UID, group 0)
RUN mkdir -p /isaac-sim/kit/data && \
    chmod -R g+w /isaac-sim/kit/cache \
                 /isaac-sim/kit/data \
                 /isaac-sim/kit/logs
```

**Why this is needed:**
- Isaac Sim writes to cache, data, and logs directories at runtime
- Without group write permissions, Isaac Sim will fail with permission errors
- All writable directories must be made accessible to group 0

### Storage Configuration

RHOAI mounts a PVC at `/opt/app-root/src` (the HOME directory). This mount **replaces any content created at build time**, so initialization must happen at runtime.

**In launch.sh, copy files to PVC on first startup:**
```bash
if [ "$OPENSHIFT_MODE" = "true" ]; then
    # Copy notebook files to PVC on first startup
    if [ ! -f /opt/app-root/src/generate_dataset.ipynb ]; then
        echo "First startup - copying notebook files to PVC..."
        cp /workspace/isaaclab/generate_dataset.ipynb /opt/app-root/src/
        cp /workspace/isaaclab/notebook_utils.py /opt/app-root/src/
        # ... copy other files
        echo "Notebook files ready"
    fi

    # Create user data directories on PVC (must be at runtime, not build time)
    echo "Creating user data directories on PVC..."
    mkdir -p /opt/app-root/src/datasets \
             /opt/app-root/src/Documents/Kit/shared

    # Copy input dataset to PVC on first startup
    if [ ! -f /opt/app-root/src/datasets/annotated_dataset.hdf5 ]; then
        echo "Copying input dataset to PVC..."
        cp /workspace/isaaclab/annotated_dataset.hdf5 /opt/app-root/src/datasets/
        echo "Input dataset ready"
    fi
fi
```

**Pattern:**
- Files are baked into the image at a non-PVC location (e.g., `/workspace/isaaclab/`)
- On first startup, launcher script checks if files exist in PVC
- If not, copies from image to PVC
- Subsequent startups skip the copy (files already present)

**Why this is needed:**
- RHOAI PVC mount at `/opt/app-root/src` overwrites any build-time content
- Users need notebooks and sample data files available on first login
- User-generated data persists across pod restarts

### GPU Resource Allocation

GPU allocation is handled by RHOAI UI when user creates a workbench. The image must be compatible with GPU nodes.

**In ImageStream, document GPU requirement:**
```yaml
annotations:
  opendatahub.io/notebook-image-desc: "Isaac Lab + Isaac Sim for robotics (GPU required)"
```

**In notebooks, configure Isaac Sim for GPU:**
```python
# Isaac Sim automatically detects GPU
# No special configuration needed in code
```

See [resource-patterns/gpu-allocation-openshift.md](../resource-patterns/gpu-allocation-openshift.md) for general GPU patterns.

### Jupyter Lab Integration

RHOAI injects `NOTEBOOK_ARGS` environment variable with correct configuration (base URL, token, password, port).

**In launch.sh:**
```bash
if [ "$OPENSHIFT_MODE" = "true" ]; then
    # Start Jupyter Lab (non-root mode for OpenShift)
    # RHOAI injects NOTEBOOK_ARGS with correct base_url, token, password, port
    /isaac-sim/python.sh -m jupyter lab \
        --ip=0.0.0.0 \
        --no-browser \
        $NOTEBOOK_ARGS

    exit 0
fi
```

**Important:**
- Use the application's Python interpreter (e.g., `/isaac-sim/python.sh`) to preserve environment
- Pass `$NOTEBOOK_ARGS` to Jupyter Lab command
- Do NOT set `--allow-root` (OpenShift runs as non-root random UID)

### Environment Variables and Config

**Required environment variables in Dockerfile:**
```dockerfile
ENV OPENSHIFT_MODE=true \
    DISPLAY=:99 \
    ACCEPT_EULA=Y \
    HOME=/opt/app-root/src
```

- `OPENSHIFT_MODE=true`: Triggers RHOAI-specific behavior
- `DISPLAY=:99`: Points to Xvfb virtual display
- `ACCEPT_EULA=Y`: Auto-accepts NVIDIA licenses (Isaac Sim, Cosmos, etc.)
- `HOME=/opt/app-root/src`: Standard RHOAI home directory (where PVC is mounted)

### Conditional Application Behavior

Applications may need to adapt based on deployment mode (e.g., using NIM API vs self-hosted inference).

**Example from cosmos_request.py:**
```python
def process_video(...):
    # Check if we should use NIM API or self-hosted API
    openshift_mode = os.getenv("OPENSHIFT_MODE", "false").lower() == "true"

    if openshift_mode:
        # Use NIM API (synchronous, /v1/infer endpoint)
        return _process_video_nim(...)

    # Direct Deployment mode (app.py wrapper with async job-based API)
    print(f"Using Direct Deployment mode (app.py wrapper)")
    # ... original logic
```

**Pattern:**
- Detect `OPENSHIFT_MODE` environment variable
- Branch logic based on mode
- RHOAI mode typically uses managed services (NIM API)
- Original mode uses self-hosted/local services

## Known Issues and Gotchas

### Issue: Isaac Sim renderer.capture fails with Xvfb

**Problem:** The `omni.kit.renderer.capture` extension causes errors when running headless with Xvfb.

**Solution:** Conditionally disable it in OpenShift mode.

```python
config = {
    "pause_subtask": False,
}

# Conditionally disable renderer.capture in OpenShift mode (causes issues with Xvfb)
openshift_mode = os.getenv("OPENSHIFT_MODE", "false").lower() == "true"
if not openshift_mode:
    config["enable"] = "omni.kit.renderer.capture"
```

### Issue: PVC mount overwrites build-time files

**Problem:** Files created in `/opt/app-root/src` during image build are lost when RHOAI mounts PVC at runtime.

**Solution:** Store files elsewhere in image (e.g., `/workspace/`) and copy to PVC on first startup (see Storage Configuration above).

### Issue: Isaac Sim writes to system directories

**Problem:** Isaac Sim writes to `/isaac-sim/kit/cache`, `/isaac-sim/kit/data`, `/isaac-sim/kit/logs` at runtime. These are read-only for group by default.

**Solution:** Make these directories writable for group 0 in Dockerfile.

```dockerfile
RUN mkdir -p /isaac-sim/kit/data && \
    chmod -R g+w /isaac-sim/kit/cache \
                 /isaac-sim/kit/data \
                 /isaac-sim/kit/logs
```

## Dependencies

- GPU-enabled OpenShift cluster
- RHOAI installed
- Namespace with appropriate GPU quotas
- PVC for user data (automatically created by RHOAI when user creates workbench)

## Testing Notes

### How to verify this component works on RHOAI

1. **Build and deploy:**
   ```bash
   ./deploy/deploy-isaac-lab-image.sh
   ```

2. **Verify ImageStream:**
   ```bash
   oc get imagestream isaac-lab -n redhat-ods-applications
   ```

3. **Create workbench in RHOAI UI:**
   - Select "Isaac Lab" notebook image
   - Choose GPU size (1 GPU minimum)
   - Create workbench

4. **Verify Xvfb is running:**
   Once workbench starts, open terminal and run:
   ```bash
   ps aux | grep Xvfb
   xdpyinfo -display :99
   ```

5. **Run the notebook:**
   - Open `generate_dataset.ipynb`
   - Execute cells
   - Verify Isaac Sim loads and renders (check for headless mode message)
   - Verify video generation completes

6. **Check data persistence:**
   - Stop workbench
   - Restart workbench
   - Verify generated datasets persist in `/opt/app-root/src/datasets/`
