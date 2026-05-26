---
type: component
components: [chromadb, sqlite, embeddings]
deployment_types: [rhoai-notebook]
resource_types: []
architecture: []
source_examples:
  - blueprint: "nvidia-demo"
    source_repo: "https://github.com/crewAIInc/nvidia-demo"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-demo"
    notes: "SQLite compatibility patch for ChromaDB in RHOAI workbenches"
    approach: "A"
---

# ChromaDB on RHOAI

## Overview

ChromaDB is a vector database commonly used for embeddings and semantic search. On RHOAI, ChromaDB requires a SQLite compatibility patch because the system SQLite version is often older than required.

**Use this pattern when:**
- Blueprint uses ChromaDB for vector storage
- Running in RHOAI workbench (Jupyter notebook)
- Encountering "SQLite version 3.35.0 or higher required" error

## Conversion Pattern

### SQLite Compatibility Patch

ChromaDB requires SQLite 3.35.0+, but RHOAI workbenches often have older versions. The solution is to replace the system `sqlite3` module with `pysqlite3-binary`.

**Implementation:**

```python
# SQLite compatibility patch (OpenShift only)
if OPENSHIFT_MODE:
    import sys
    __import__('pysqlite3')
    sys.modules['sqlite3'] = sys.modules.pop('pysqlite3')
    print("✓ SQLite patch applied (using pysqlite3-binary)")
```

**Key points:**
- Must run **before** any ChromaDB imports
- Only needed in RHOAI mode (system SQLite is fine in docker-compose)
- Uses module substitution to transparently replace sqlite3

### Dependency Installation

Install `pysqlite3-binary` package:

```python
if OPENSHIFT_MODE:
    %pip install -r requirements.txt pysqlite3-binary>=0.5.4 -q
else:
    %pip install -r requirements.txt -q
```

Or add to `requirements.txt`:
```
pysqlite3-binary>=0.5.4
```

### Complete Pattern (Notebook Cell)

Place this early in your notebook, before any ChromaDB/CrewAI imports:

```python
import os

# Detect deployment mode
OPENSHIFT_MODE = os.getenv('OPENSHIFT_MODE', 'false').lower() == 'true'

# SQLite compatibility patch (must run before ChromaDB imports)
if OPENSHIFT_MODE:
    import sys
    __import__('pysqlite3')
    sys.modules['sqlite3'] = sys.modules.pop('pysqlite3')
    print("✓ SQLite patch applied (using pysqlite3-binary)")

# Now safe to install packages that depend on ChromaDB
if OPENSHIFT_MODE:
    %pip install -r requirements.txt pysqlite3-binary>=0.5.4 -q
else:
    %pip install -r requirements.txt -q
```

## Environment Variables and Config

No environment variables needed - the patch is automatically applied when `OPENSHIFT_MODE=true`.

## Known Issues and Gotchas

### Issue: "SQLite version 3.35.0 or higher required"

**Cause:** Patch not applied or applied after ChromaDB import.

**Solution:** 
1. Ensure `OPENSHIFT_MODE=true` is set
2. Move SQLite patch to top of notebook (before pip install)
3. Restart kernel and re-run from beginning

### Issue: "No module named 'pysqlite3'"

**Cause:** `pysqlite3-binary` package not installed.

**Solution:**
```bash
pip install pysqlite3-binary>=0.5.4
```

### Issue: Patch applied but still getting SQLite error

**Cause:** ChromaDB was imported before the patch.

**Solution:** 
1. Restart kernel
2. Ensure patch cell runs first
3. Verify import order:
   ```python
   # ✓ Correct order
   import sys
   __import__('pysqlite3')
   sys.modules['sqlite3'] = sys.modules.pop('pysqlite3')
   
   # Now safe to import ChromaDB-dependent packages
   from chromadb import ...
   from crewai_tools import WebsiteSearchTool  # Uses ChromaDB internally
   ```

### Issue: "Database is locked" error

**Cause:** Multiple ChromaDB instances accessing the same database file.

**Solution:** 
1. Ensure only one ChromaDB instance per database
2. Close/delete previous instances before creating new ones
3. In RHOAI, databases are in `/opt/app-root/src` (PVC) - persist across pod restarts

## Dependencies

- `pysqlite3-binary>=0.5.4` - Drop-in replacement for sqlite3 with newer version
- `chromadb` - Vector database (specified in blueprint's requirements.txt)

## Testing Notes

Verify the patch works:

```python
import os
os.environ['OPENSHIFT_MODE'] = 'true'

# Apply patch
import sys
__import__('pysqlite3')
sys.modules['sqlite3'] = sys.modules.pop('pysqlite3')

# Check SQLite version
import sqlite3
print(f"SQLite version: {sqlite3.sqlite_version}")  # Should be >= 3.35.0

# Test ChromaDB
import chromadb
client = chromadb.Client()
print("✓ ChromaDB initialized successfully")
```

## Why This Is Needed

ChromaDB uses DuckDB internally, which requires SQLite 3.35.0+ for certain features. RHOAI workbench base images often include older SQLite versions (e.g., 3.32.x) from the OS package manager.

Rather than rebuilding the entire workbench image with a newer SQLite, we use `pysqlite3-binary` which:
- Provides a self-contained SQLite implementation
- Compiles SQLite from source during pip install
- Guarantees SQLite 3.35.0+
- Drops in as a replacement for the `sqlite3` module

The module substitution (`sys.modules['sqlite3'] = ...`) makes Python import the newer version whenever any code tries to `import sqlite3`.

## Related Patterns

- [crewai-on-rhoai.md](crewai-on-rhoai.md) - CrewAI uses ChromaDB for embeddings
- [deployment-types/rhoai-notebook-pattern.md](../deployment-types/rhoai-notebook-pattern.md) - Notebook deployment context
