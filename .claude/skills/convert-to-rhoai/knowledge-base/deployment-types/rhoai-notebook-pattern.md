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
  - blueprint: "generative-protein-binder-design"
    source_repo: "https://github.com/NVIDIA-BioNeMo-blueprints/generative-protein-binder-design"
    fork_repo: "https://github.com/rh-ai-quickstart/generative-protein-binder-design"
    notes: "DEPLOYMENT_MODE environment variable pattern for switching between localhost and Kubernetes DNS names, with connection keepalive thread"
    approach: "A"
  - blueprint: "nvidia-demo"
    source_repo: "https://github.com/crewAIInc/nvidia-demo"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-demo"
    notes: "Standard workbench approach - no custom Dockerfile needed, OPENSHIFT_MODE in notebook cells to switch between NVIDIA API Catalog and self-hosted NIMs"
    approach: "B"
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

### 5. Update Notebooks for DEPLOYMENT_MODE

**Purpose:** Notebooks may need to adapt behavior based on deployment mode.

**Two common patterns:**

#### Pattern A: OPENSHIFT_MODE (boolean flag)

Used when notebooks need to enable/disable features:

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

#### Pattern B: DEPLOYMENT_MODE (mode string)

Used when notebooks need to switch service URLs between local and cluster-internal:

```python
import os

DEPLOYMENT_MODE = os.getenv('DEPLOYMENT_MODE', 'local')

if DEPLOYMENT_MODE == 'openshift':
    # Cluster-internal K8s DNS names
    NIM_HOSTS = {
        'SERVICE1': os.getenv('SERVICE1_HOST', 'http://service1'),
        'SERVICE2': os.getenv('SERVICE2_HOST', 'http://service2'),
    }
    PORTS = {
        'SERVICE1': 8000,  # All services use 8000 on OpenShift
        'SERVICE2': 8000,
    }
    print("OpenShift/RHOAI Mode - Using Helm-deployed services")
else:
    # Localhost URLs for docker-compose
    NIM_HOSTS = {
        'SERVICE1': 'http://localhost',
        'SERVICE2': 'http://localhost',
    }
    PORTS = {
        'SERVICE1': 8081,  # Different ports per service locally
        'SERVICE2': 8082,
    }
    print("Local Mode - Using localhost")
```

**Example from generative-protein-binder-design:**
```python
# RECOMMENDED: Set DEPLOYMENT_MODE=openshift when creating your RHOAI workbench.
# FALLBACK: Uncomment the line below if you forgot to set it during creation.
# os.environ['DEPLOYMENT_MODE'] = 'openshift'

DEPLOYMENT_MODE = os.getenv('DEPLOYMENT_MODE', 'local')

if DEPLOYMENT_MODE == 'openshift':
    # Cluster-internal K8s DNS names. Defaults match NIM Operator service names.
    # Override with env vars if using standard Helm chart service names.
    NIM_HOSTS = {
        'ALPHAFOLD2': os.getenv('ALPHAFOLD2_HOST', 'http://alphafold2'),
        'RFDIFFUSION': os.getenv('RFDIFFUSION_HOST', 'http://rfdiffusion'),
        'PROTEINMPNN': os.getenv('PROTEINMPNN_HOST', 'http://proteinmpnn'),
        'AF2_MULTIMER': os.getenv('AF2_MULTIMER_HOST', 'http://alphafold2-multimer'),
    }
    print("OpenShift/RHOAI Mode - Using Helm-deployed NIM services")
else:
    NIM_HOSTS = {name: 'http://localhost' for name in ['ALPHAFOLD2', 'RFDIFFUSION', 'PROTEINMPNN', 'AF2_MULTIMER']}
    print("Local Mode - Using localhost (deploy with docker compose first)")

# Port mapping: localhost uses different ports per service, OpenShift uses 8000 for all
_OCP_PORT = 8000
if DEPLOYMENT_MODE == 'openshift':
    NIM_PORT_TO_BASE_URL = {
        8081: f"{NIM_HOSTS['ALPHAFOLD2']}:{_OCP_PORT}",
        8082: f"{NIM_HOSTS['RFDIFFUSION']}:{_OCP_PORT}",
        8083: f"{NIM_HOSTS['PROTEINMPNN']}:{_OCP_PORT}",
        8084: f"{NIM_HOSTS['AF2_MULTIMER']}:{_OCP_PORT}",
    }
else:
    NIM_PORT_TO_BASE_URL = {
        8081: f"{NIM_HOSTS['ALPHAFOLD2']}:8081",
        8082: f"{NIM_HOSTS['RFDIFFUSION']}:8082",
        8083: f"{NIM_HOSTS['PROTEINMPNN']}:8083",
        8084: f"{NIM_HOSTS['AF2_MULTIMER']}:8084",
    }

print(f"\nService URLs:")
for name in NIM_HOSTS:
    port = 8081  # lookup actual port for this service
    print(f"  {name}: {NIM_PORT_TO_BASE_URL[port]}")
```

**Key differences between patterns:**
- **OPENSHIFT_MODE (boolean)**: Use for feature flags (headless mode, GPU rendering, etc.)
- **DEPLOYMENT_MODE (string)**: Use for URL/service discovery (localhost vs Kubernetes DNS)
- Both can be used in the same notebook if needed

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

### Issue: Long-running notebook cells timeout in RHOAI

**Problem:** When running long-running cells (e.g., 20+ minute API calls to NIMs), the RHOAI connection may timeout and show "kernel connection lost" even though the cell is still executing on the backend.

**Solution:** Add a keepalive thread that periodically prints to prevent connection timeout:

**Example from generative-protein-binder-design:**
```python
if DEPLOYMENT_MODE == 'openshift':
    import threading, time as _time
    def _keepalive():
        while True:
            _time.sleep(60)
            print('.', end='', flush=True)
    threading.Thread(target=_keepalive, daemon=True).start()
    print('Keepalive thread started (prints a dot every 60s to prevent connection timeout)')
```

**Pattern details:**
- Start a daemon thread that prints `.` every 60 seconds
- Only enable in OpenShift mode (not needed for local docker-compose)
- Use `flush=True` to ensure output is sent immediately
- Daemon thread dies when notebook kernel stops (no cleanup needed)
- Minimal output keeps connection alive without cluttering cell output

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

---

## Approach B: Standard Workbench (No Custom Image) - from nvidia-demo

### When to Use

Use this approach when:
- Blueprint doesn't require custom system dependencies or GPU drivers
- Can run in standard RHOAI workbench images (e.g., Jupyter Minimal Python)
- Only needs Python package installation
- Switches between external APIs (NVIDIA API Catalog) and self-hosted services (NIMs)
- No special entrypoint logic needed

**Don't use this approach when:**
- Need custom system packages (e.g., Xvfb, CUDA drivers)
- Need to modify entrypoint or startup sequence
- Need to bake data files into the image
- Require specific base images (e.g., Isaac Lab, BioNeMo)

### Differences from Approach A

| Aspect | Approach A (Custom Image) | Approach B (Standard Workbench) |
|--------|---------------------------|--------------------------------|
| **Base Image** | Custom Dockerfile extending NVIDIA images | Standard RHOAI workbench image |
| **Dependencies** | Baked into image at build time | Installed at runtime via pip |
| **OPENSHIFT_MODE** | Set in Dockerfile ENV | Set in workbench environment variables |
| **Entrypoint** | Custom launch.sh with PVC initialization | Standard Jupyter Lab (no custom script) |
| **Deployment** | BuildConfig + ImageStream | Just create workbench and run notebook |
| **Complexity** | Higher - requires build/deploy scripts | Lower - no build step needed |
| **Use Cases** | Complex environments (GPU drivers, system deps) | Pure Python workflows |

### Conversion Steps

#### 1. Identify OPENSHIFT_MODE Usage

Add environment detection at the top of the notebook:

```python
import os

# Detect deployment mode
OPENSHIFT_MODE = os.getenv('OPENSHIFT_MODE', 'false').lower() == 'true'

if OPENSHIFT_MODE:
    print("🔧 OpenShift Mode: Enabled (self-hosted NVIDIA NIMs)")
else:
    print("🌐 Standard Mode: Using NVIDIA API Catalog")
```

#### 2. Add Dependency Installation

Install dependencies conditionally based on mode:

```python
# Install dependencies
loader = LoadingAnimation()  # Optional: visual feedback
loader.start("Installing")

if OPENSHIFT_MODE:
    # RHOAI-specific dependencies
    %pip install -r requirements.txt pysqlite3-binary>=0.5.4 -q
else:
    # Standard dependencies
    %pip install -r requirements.txt -q

loader.stop("Installation complete")
```

**Common RHOAI-specific dependencies:**
- `pysqlite3-binary` - SQLite compatibility for ChromaDB (see [chromadb-on-rhoai.md](../components/chromadb-on-rhoai.md))

#### 3. Add Environment Variable Collection

Prompt for configuration if not pre-set:

```python
import getpass

if OPENSHIFT_MODE:
    # Self-hosted NIM endpoints
    if not os.environ.get("OPENAI_API_BASE"):
        endpoint = input("Enter self-hosted NIM endpoint (e.g., https://llama-project.apps.cluster.com/v1): ")
        os.environ["OPENAI_API_BASE"] = endpoint
    
    if not os.environ.get("OPENAI_API_KEY"):
        nim_token = getpass.getpass("Enter NIM token: ")
        os.environ["OPENAI_API_KEY"] = nim_token
else:
    # API Catalog key
    if not os.environ.get("NVIDIA_NIM_API_KEY", "").startswith("nvapi-"):
        nvapi_key = getpass.getpass("Enter NVIDIA API key: ")
        os.environ["NVIDIA_NIM_API_KEY"] = nvapi_key
```

#### 4. Modify Service Configuration

Switch service endpoints based on mode:

**For LLM services (e.g., CrewAI):**
```python
from crewai import LLM

if OPENSHIFT_MODE:
    llm = LLM(
        model="openai/meta/llama-3.1-8b-instruct",
        base_url=os.getenv('OPENAI_API_BASE'),
        api_key=os.getenv('OPENAI_API_KEY')
    )
else:
    llm = LLM(model="nvidia_nim/meta/llama-3.3-70b-instruct")
```

**For embeddings:**
```python
if OPENSHIFT_MODE:
    embedder_config = dict(
        provider="openai",
        config=dict(
            model="nvidia/nv-embedqa-e5-v5-passage",
            api_base=os.environ.get("NVIDIA_EMBED_BASE_URL"),
            api_key=os.environ.get("NVIDIA_EMBED_TOKEN")
        )
    )
else:
    embedder_config = dict(
        provider="nvidia",
        config=dict(model="nvidia/nv-embedqa-e5-v5")
    )
```

See [components/crewai-on-rhoai.md](../components/crewai-on-rhoai.md) for detailed CrewAI patterns.

#### 5. Update README

Add RHOAI deployment section:

```markdown
## Self-Hosted Deployment on RHOAI

On **OpenShift AI**, work through your **data science project** in this order: 
**deploy the LLM and embedding NIMs** (Part 1) → **create a workbench** and set 
environment variables (Part 2) → **start the workbench** → **clone this repository** 
into the workspace → **open and run** `<notebook>.ipynb`.

### Part 1: Deploy NVIDIA NIMs

In a project with **NVIDIA NIM** enabled and sufficient **GPU**:

1. **LLM deployment**: 
   - Models → NVIDIA NIM → Deploy model → `meta/llama-3.1-8b-instruct`
   - Resources: 1 GPU, 16GiB memory, 4 CPU
   - Enable token auth
   - Copy inference URL (`/v1` endpoint) → `OPENAI_API_BASE`
   - Copy token → `OPENAI_API_KEY`

2. **Embedding NIM**:
   - Deploy `nvidia/nv-embedqa-e5-v5`
   - Same resources as LLM
   - Copy URL → `NVIDIA_EMBED_BASE_URL`
   - Copy token → `NVIDIA_EMBED_TOKEN`

### Part 2: Workbench Setup

1. **Create workbench** in the project:
   - Image: Jupyter Minimal Python 3.12 (or similar)
   - Container size: Small
   - Accelerator: None (NIMs use GPUs, notebook doesn't)
   
2. **Add environment variables**:

   | Variable | Value |
   |----------|-------|
   | `OPENSHIFT_MODE` | `true` |
   | `OPENAI_API_BASE` | `https://your-llm-endpoint/v1` |
   | `OPENAI_API_KEY` | `your_llm_token` |
   | `NVIDIA_EMBED_BASE_URL` | `https://your-embed-endpoint/v1` |
   | `NVIDIA_EMBED_TOKEN` | `your_embed_token` |

3. **Start workbench**, clone repo, open notebook, run cells.
```

### Known Issues (Approach B Specific)

#### Issue: "SQLite version too old" when using ChromaDB

**Solution:** Apply SQLite compatibility patch (see [chromadb-on-rhoai.md](../components/chromadb-on-rhoai.md)):

```python
if OPENSHIFT_MODE:
    import sys
    __import__('pysqlite3')
    sys.modules['sqlite3'] = sys.modules.pop('pysqlite3')
```

#### Issue: Notebook cells not prompting for credentials

**Cause:** Environment variables already set in workbench but values are wrong.

**Solution:** 
1. Edit workbench → Environment variables
2. Update values
3. Restart workbench
4. Alternatively, override in notebook:
   ```python
   os.environ["OPENAI_API_BASE"] = "https://correct-endpoint/v1"
   ```

#### Issue: Import errors for packages

**Cause:** Dependencies not installed in workbench.

**Solution:** 
1. Ensure pip install cell runs successfully
2. Check for errors in pip output
3. Restart kernel if needed
4. Manually install if needed: `!pip install <package>`

---

## Choosing Between Approaches

### Use Approach A (Custom Image) when:
- Blueprint requires system dependencies (Xvfb, custom drivers, system libraries)
- Blueprint extends specific NVIDIA base images (Isaac Lab, BioNeMo)
- Need to bake large datasets into the image
- Need custom entrypoint logic (PVC initialization, service startup)
- Blueprint architecture requires specific runtime environment

**Examples:**
- Isaac Lab blueprints (need GPU drivers, Xvfb, Isaac Sim)
- BioNeMo blueprints (need specific base images)
- Blueprints with large pre-installed datasets

### Use Approach B (Standard Workbench) when:
- Blueprint is pure Python (no system dependencies)
- Can run in standard Jupyter environments
- Only switches between API endpoints (NVIDIA API Catalog vs self-hosted NIMs)
- Minimal setup required
- Primarily notebook-based interaction

**Examples:**
- CrewAI agentic workflows (nvidia-demo)
- Pure Python ML/AI workflows
- API-based blueprints without local services

### Decision Tree

```
Does blueprint require custom system packages or GPU drivers?
├─ YES → Use Approach A (Custom Image)
└─ NO → Does blueprint need specific NVIDIA base images?
         ├─ YES → Use Approach A (Custom Image)
         └─ NO → Can it run in standard Python environment?
                  ├─ YES → Use Approach B (Standard Workbench)
                  └─ NO → Use Approach A (Custom Image)
```

## Related Patterns

- [components/isaac-lab-on-rhoai.md](../components/isaac-lab-on-rhoai.md) - Example Approach A implementation
- [components/crewai-on-rhoai.md](../components/crewai-on-rhoai.md) - Example Approach B implementation
- [components/chromadb-on-rhoai.md](../components/chromadb-on-rhoai.md) - SQLite compatibility (Approach B)
- [resource-patterns/gpu-allocation-openshift.md](../resource-patterns/gpu-allocation-openshift.md) - GPU configuration
- [resource-patterns/security-contexts-scc.md](../resource-patterns/security-contexts-scc.md) - Security contexts
