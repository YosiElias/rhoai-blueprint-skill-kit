---
type: resource-pattern
components: []
deployment_types: [rhoai-notebook]
resource_types: [storage]
architecture: []
source_examples:
  - blueprint: "synthetic-manipulation-motion-generation"
    source_repo: "https://github.com/NVIDIA-Omniverse-blueprints/synthetic-manipulation-motion-generation"
    fork_repo: "https://github.com/rh-ai-quickstart/synthetic-manipulation-motion-generation"
    notes: "Demonstrates runtime PVC initialization pattern for RHOAI notebooks"
    approach: "A"
---

# RHOAI PVC Initialization Pattern

## Overview

Red Hat OpenShift AI (RHOAI) mounts a Persistent Volume Claim (PVC) at `/opt/app-root/src` (the container's HOME directory) when launching notebook workbenches. This mount **replaces any content that was created at that location during the Docker build**. This means:

- Files created in `/opt/app-root/src` during `docker build` are **lost** at runtime
- Directories created in `/opt/app-root/src` during `docker build` are **lost** at runtime
- The PVC starts empty on first launch

This pattern shows how to initialize the PVC with required files and directories at runtime, not build time.

## When to Use

- When deploying custom RHOAI notebook images
- When notebooks need starter files (notebooks, scripts, sample data)
- When the application needs specific directory structure in HOME
- When input data files need to be accessible to the notebook user

## The Problem

**What doesn't work:**

```dockerfile
# In Dockerfile
WORKDIR /opt/app-root/src
COPY notebook.ipynb /opt/app-root/src/
COPY samples/*.hdf5 /opt/app-root/src/datasets/
RUN mkdir -p /opt/app-root/src/output
```

**Why it doesn't work:**
- RHOAI mounts PVC at `/opt/app-root/src` at runtime
- The mount **replaces** the filesystem at that path
- All files copied during build are hidden by the mount
- User sees an empty directory on first login

## The Solution

### 1. Store Files Outside PVC at Build Time

In Dockerfile, copy files to a location that is **not** `/opt/app-root/src`:

```dockerfile
# Copy files to a non-PVC location
COPY notebook/*.ipynb /workspace/notebooks/
COPY notebook/*.py /workspace/notebooks/
COPY samples/*.hdf5 /workspace/data/

# Do NOT copy to /opt/app-root/src - those files will be lost!
```

**Common locations:**
- `/workspace/`
- `/app/`
- `/opt/<app-name>/`
- Application-specific paths (e.g., `/isaac-sim/`, `/bitnami/`)

### 2. Copy to PVC at Runtime

In the launcher script (entrypoint), check if this is the first startup and copy files to PVC:

```bash
#!/bin/bash
set -e

if [ "$OPENSHIFT_MODE" = "true" ]; then
    # Copy notebook files to PVC on first startup
    if [ ! -f /opt/app-root/src/main_notebook.ipynb ]; then
        echo "First startup - copying notebook files to PVC..."
        cp /workspace/notebooks/*.ipynb /opt/app-root/src/
        cp /workspace/notebooks/*.py /opt/app-root/src/
        echo "Notebook files ready"
    fi

    # Create user data directories on PVC (must be at runtime, not build time)
    echo "Creating user data directories on PVC..."
    mkdir -p /opt/app-root/src/datasets \
             /opt/app-root/src/output \
             /opt/app-root/src/logs

    # Copy input dataset to PVC on first startup
    if [ ! -f /opt/app-root/src/datasets/input_data.hdf5 ]; then
        echo "Copying input dataset to PVC..."
        cp /workspace/data/input_data.hdf5 /opt/app-root/src/datasets/
        echo "Input dataset ready"
    fi

    # Launch Jupyter Lab...
fi
```

**Pattern explanation:**
1. Check if marker file exists (`main_notebook.ipynb`)
2. If not, this is first startup - copy files from image to PVC
3. Create any required directory structure
4. Copy large input data files only on first startup
5. Subsequent startups skip the copy (files already present)

### 3. Set Permissions in Dockerfile

Even though files are stored outside PVC, you still need to set group 0 permissions:

```dockerfile
# Make source directories readable by group 0 (in case user needs to re-copy)
RUN chgrp -R 0 /workspace && \
    chmod -R g+rx /workspace

# Ensure /opt/app-root/src has correct ownership (even though it will be replaced by PVC)
RUN chgrp -R 0 /opt/app-root/src && \
    chmod -R g=u /opt/app-root/src
```

**Why:**
- OpenShift runs as random UID, group 0
- Files must be readable by group 0 for the copy to work
- `/opt/app-root/src` permissions are set for consistency (though PVC will replace it)

## Complete Example

**Dockerfile:**
```dockerfile
FROM nvcr.io/nvidia/base-image:latest

# Install dependencies
RUN apt-get update && apt-get install -y python3-pip

# Set RHOAI environment variables
ENV OPENSHIFT_MODE=true \
    HOME=/opt/app-root/src

# Install Jupyter Lab
RUN python3 -m pip install jupyter

# Copy files to NON-PVC location
COPY notebook/main.ipynb /workspace/notebooks/main.ipynb
COPY notebook/utils.py /workspace/notebooks/utils.py
COPY samples/dataset.hdf5 /workspace/data/dataset.hdf5
COPY launch.sh /workspace/launch.sh

# Set permissions for group 0
RUN chgrp -R 0 /workspace && \
    chmod -R g+rx /workspace && \
    chmod +x /workspace/launch.sh

# Permissions for PVC directory (will be replaced by mount, but set for consistency)
RUN mkdir -p /opt/app-root/src && \
    chgrp -R 0 /opt/app-root/src && \
    chmod -R g=u /opt/app-root/src

WORKDIR /opt/app-root/src
ENTRYPOINT ["/workspace/launch.sh"]
```

**launch.sh:**
```bash
#!/bin/bash
set -e

if [ "$OPENSHIFT_MODE" = "true" ]; then
    echo "Running in OpenShift mode"

    # First startup: copy notebook files to PVC
    if [ ! -f /opt/app-root/src/main.ipynb ]; then
        echo "First startup - copying notebook files to PVC..."
        cp /workspace/notebooks/*.ipynb /opt/app-root/src/
        cp /workspace/notebooks/*.py /opt/app-root/src/
        echo "Notebook files ready"
    fi

    # Always create directory structure (in case user deleted them)
    echo "Creating user data directories on PVC..."
    mkdir -p /opt/app-root/src/datasets \
             /opt/app-root/src/output

    # First startup: copy input data to PVC
    if [ ! -f /opt/app-root/src/datasets/dataset.hdf5 ]; then
        echo "Copying input dataset to PVC..."
        cp /workspace/data/dataset.hdf5 /opt/app-root/src/datasets/
        echo "Input dataset ready"
    fi

    # Start Jupyter Lab
    python3 -m jupyter lab \
        --ip=0.0.0.0 \
        --no-browser \
        $NOTEBOOK_ARGS

    exit 0
fi

# Original docker-compose mode
# ...
```

## Design Decisions

### Why use a marker file check?

**Option 1: Always copy on startup**
```bash
# Don't do this
cp /workspace/notebooks/*.ipynb /opt/app-root/src/
```
**Problem:** Overwrites user modifications on every pod restart.

**Option 2: Check for marker file**
```bash
# Do this
if [ ! -f /opt/app-root/src/main.ipynb ]; then
    cp /workspace/notebooks/*.ipynb /opt/app-root/src/
fi
```
**Benefit:** Preserves user modifications, copies only on first startup.

### What file to use as marker?

Use the **primary notebook file** as the marker:
```bash
if [ ! -f /opt/app-root/src/main_notebook.ipynb ]; then
```

**Why:**
- It's unlikely to be deleted by user
- If user deletes it, re-copying is desired behavior
- Clear indication that initialization hasn't happened

**Don't use:**
- Hidden files (`.initialized`) - user may not notice if deleted
- Data files - user might intentionally delete/replace them
- Directories - can be created accidentally

### Should directories be created on every startup?

**Yes, recreate directories on every startup:**
```bash
# Always do this
mkdir -p /opt/app-root/src/datasets \
         /opt/app-root/src/output
```

**Why:**
- `mkdir -p` is idempotent (safe to run multiple times)
- If user accidentally deletes a directory, it's recreated
- Ensures expected directory structure always exists

### Should large data files be copied or mounted?

**For input data < 1GB:** Copy at runtime (pattern above)

**For large datasets > 1GB:** Consider alternatives:
1. **Object storage:** Mount S3/MinIO bucket, reference URLs in notebook
2. **Separate PVC:** Create a second PVC for read-only datasets
3. **DownloadURL annotation:** Use RHOAI's download mechanism (if supported)

**Don't:** Embed large datasets in Docker image (bloats image size).

## Known Issues and Gotchas

### Issue: Files disappear after pod restart

**Cause:** Files were created in ephemeral storage, not PVC.

**Solution:** Ensure files are in `/opt/app-root/src` (the PVC mount point).

### Issue: First startup is slow

**Cause:** Large files being copied from image to PVC.

**Solution:** 
- For large datasets, use object storage instead
- Display progress during copy: `echo "Copying 1.5GB dataset (may take 2-3 minutes)..."`

### Issue: User modifications lost on pod restart

**Cause:** Launcher script copies files on every startup, overwriting changes.

**Solution:** Use marker file check (see "Complete Example" above).

### Issue: Directory permissions wrong after creation

**Cause:** `mkdir` creates directories with default permissions (typically 755).

**Solution:** Set permissions after creation:
```bash
mkdir -p /opt/app-root/src/datasets
chmod -R g+w /opt/app-root/src/datasets
```

Or use `-m` flag:
```bash
mkdir -p -m 775 /opt/app-root/src/datasets
```

### Issue: Copy fails with "permission denied"

**Cause:** Source files not readable by group 0.

**Solution:** In Dockerfile, set group 0 read permissions:
```dockerfile
RUN chgrp -R 0 /workspace && chmod -R g+rx /workspace
```

## Verification

After deploying, verify the pattern works:

1. **Create workbench and start**
2. **Check files are present on first startup:**
   ```bash
   ls -la /opt/app-root/src/
   ls -la /opt/app-root/src/datasets/
   ```
3. **Modify a notebook, add cells, save**
4. **Stop workbench**
5. **Restart workbench**
6. **Verify modifications persisted:**
   - Open the notebook
   - Check that your added cells are still there

## Related Patterns

- [deployment-types/rhoai-notebook-pattern.md](../deployment-types/rhoai-notebook-pattern.md) - Full RHOAI notebook deployment pattern
- [components/isaac-lab-on-rhoai.md](../components/isaac-lab-on-rhoai.md) - Example implementation
- [resource-patterns/security-contexts-scc.md](./security-contexts-scc.md) - Group 0 permissions context
