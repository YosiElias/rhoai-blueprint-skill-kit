---
type: component
components: [crewai, llm]
deployment_types: [rhoai-notebook]
resource_types: []
architecture: [agentic]
source_examples:
  - blueprint: "nvidia-demo"
    source_repo: "https://github.com/crewAIInc/nvidia-demo"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-demo"
    notes: "CrewAI agentic workflow with OPENSHIFT_MODE switching between NVIDIA API Catalog and self-hosted NIMs"
    approach: "A"
---

# CrewAI on RHOAI

## Overview

CrewAI is a framework for building multi-agent AI systems. This pattern covers adapting CrewAI-based blueprints to run on RHOAI with self-hosted NVIDIA NIM models instead of the NVIDIA API Catalog.

**Use this pattern when:**
- Blueprint uses CrewAI framework
- Original blueprint uses NVIDIA API Catalog for LLM access
- Need to switch to self-hosted NIMs deployed via RHOAI NIM operator
- Blueprint is notebook-based

## Conversion Pattern

### OPENSHIFT_MODE Conditional Support

Use `OPENSHIFT_MODE` environment variable to switch between deployment modes:
- **false/unset** (default): NVIDIA API Catalog (hosted models)
- **true**: Self-hosted NIMs on RHOAI (OpenAI-compatible endpoints)

**Implementation in notebook:**

```python
import os

# Detect deployment mode
OPENSHIFT_MODE = os.getenv('OPENSHIFT_MODE', 'false').lower() == 'true'

if OPENSHIFT_MODE:
    print("🔧 OpenShift Mode: Enabled (self-hosted NVIDIA NIMs)")
    print("   Using self-hosted endpoints with OpenAI-compatible API")
else:
    print("🌐 Standard Mode: Using NVIDIA API Catalog")
    print("   Model: Llama 3.3 70B")
```

### LLM Configuration

#### Standard Mode (NVIDIA API Catalog)

```python
from crewai import LLM

llm = LLM(model="nvidia_nim/meta/llama-3.3-70b-instruct")
```

Environment variables required:
- `NVIDIA_NIM_API_KEY`: API key starting with `nvapi-`

#### RHOAI Mode (Self-Hosted NIMs)

```python
from crewai import LLM

llm = LLM(
    model="openai/meta/llama-3.1-8b-instruct",  # Note: openai/ provider prefix
    base_url=os.getenv('OPENAI_API_BASE'),      # NIM inference endpoint
    api_key=os.getenv('OPENAI_API_KEY')         # NIM token
)
```

Environment variables required:
- `OPENAI_API_BASE`: NIM inference endpoint URL (e.g., `https://llama-project.apps.cluster.com/v1`)
- `OPENAI_API_KEY`: NIM authentication token

**Key points:**
- Use `openai/` provider prefix for self-hosted NIMs (they expose OpenAI-compatible API)
- Model name after prefix must match the NIM model ID (e.g., `meta/llama-3.1-8b-instruct`)
- Endpoint URL must end with `/v1`

### Environment Variable Collection Pattern

Prompt users for credentials if not already set:

```python
import getpass
import os

if OPENSHIFT_MODE:
    print("🔧 Configuring for self-hosted NVIDIA NIM models...")
    
    # LLM endpoint
    if not os.environ.get("OPENAI_API_BASE"):
        endpoint = input("Enter your self-hosted NIM endpoint URL (e.g., https://llama-project.apps.cluster.com/v1): ")
        os.environ["OPENAI_API_BASE"] = endpoint
    
    if not os.environ.get("OPENAI_API_KEY"):
        nim_token = getpass.getpass("Enter your self-hosted NIM token: ")
        os.environ["OPENAI_API_KEY"] = nim_token
    
    # Embedding endpoint (separate from LLM)
    if not os.environ.get("NVIDIA_EMBED_BASE_URL"):
        embed_endpoint = input("Enter your self-hosted embedding endpoint URL: ")
        os.environ["NVIDIA_EMBED_BASE_URL"] = embed_endpoint
    
    if not os.environ.get("NVIDIA_EMBED_TOKEN"):
        embed_token = getpass.getpass("Enter your embedding token: ")
        os.environ["NVIDIA_EMBED_TOKEN"] = embed_token
else:
    # NVIDIA API Catalog
    if not os.environ.get("NVIDIA_NIM_API_KEY", "").startswith("nvapi-"):
        nvapi_key = getpass.getpass("Enter your NVIDIA API key: ")
        assert nvapi_key.startswith("nvapi-"), f"{nvapi_key[:5]}... is not a valid key"
        os.environ["NVIDIA_NIM_API_KEY"] = nvapi_key
        os.environ["NVIDIA_API_KEY"] = nvapi_key
```

**Pattern benefits:**
- Environment variables can be set in RHOAI workbench configuration (recommended)
- Fallback to interactive prompts if not pre-configured
- Validation for API key format
- Secure password input using `getpass`

### Agent Configuration Override

Externalize agent LLM configurations to YAML files for easy model switching:

**Agent config file (`config/documentation_agents.yaml`):**

```yaml
overview_writer:
  role: >
    Overview Documentation Writer
  goal: >
    Create clear, comprehensive high-level documentation
  backstory: >
    You are a technical writer specialized in creating project overviews
  verbose: false
  llm: nvidia_nim/meta/llama-3.3-70b-instruct  # Default: API Catalog

documentation_reviewer:
  role: >
    Documentation Quality Reviewer
  goal: >
    Review and ensure consistency of documentation
  backstory: >
    You are a documentation quality expert
  verbose: false
  llm: nvidia_nim/meta/llama-3.3-70b-instruct  # Default: API Catalog
```

**Override function in notebook:**

```python
import yaml
import os

def override_agent_llm_config(agents_config):
    """Override LLM configuration for OpenShift mode"""
    if OPENSHIFT_MODE:
        llm_provider = os.getenv('LLM_PROVIDER', 'openai')
        llm_model = os.getenv('LLM_MODEL', 'meta/llama-3.1-8b-instruct')
        llm_config = f"{llm_provider}/{llm_model}"
        
        # Override all agent LLM configs
        for agent_name in agents_config:
            agents_config[agent_name]['llm'] = llm_config
        
        print(f"✓ Agent configs overridden: {llm_config}")
    return agents_config

# Load and override configs
with open('config/planner_agents.yaml', 'r') as f:
    agents_config = yaml.safe_load(f)

agents_config = override_agent_llm_config(agents_config)

# Create agents
from crewai import Agent

code_explorer = Agent(
    config=agents_config['code_explorer'],
    tools=[...]
)
```

**Benefits:**
- Change models without editing notebook code
- Environment variable override for flexibility (`LLM_PROVIDER`, `LLM_MODEL`)
- Same YAML config works for both modes (override happens at runtime)

### Embedding Configuration

CrewAI tools (like `WebsiteSearchTool`) support embeddings. Configuration differs by mode:

#### Standard Mode (NVIDIA API Catalog)

```python
from crewai_tools import WebsiteSearchTool

website_search_tool = WebsiteSearchTool(
    website="https://mermaid.js.org/intro/",
    config=dict(
        embedder=dict(
            provider="nvidia",
            config=dict(model="nvidia/nv-embedqa-e5-v5")
        )
    )
)
```

#### RHOAI Mode (Self-Hosted NIM)

```python
from crewai_tools import WebsiteSearchTool

website_search_tool = WebsiteSearchTool(
    website="https://mermaid.js.org/intro/",
    config=dict(
        embedder=dict(
            provider="openai",
            config=dict(
                model="nvidia/nv-embedqa-e5-v5-passage",  # Note: -passage suffix for indexing
                api_base=os.environ.get("NVIDIA_EMBED_BASE_URL"),
                api_key=os.environ.get("NVIDIA_EMBED_TOKEN")
            )
        )
    )
)
```

**Key differences:**
- Provider changes from `nvidia` to `openai`
- Model name includes mode suffix: `-passage` for indexing, `-query` for searching
- Separate endpoint and token from LLM endpoint

### Embedding Mode Switching (RHOAI Only)

E5 embedding models require different modes for indexing vs. searching:
- **passage mode**: Use when indexing/embedding documents
- **query mode**: Use when searching/retrieving

**Implementation:**

```python
# Embedding mode switch helper (OpenShift only)
if OPENSHIFT_MODE:
    def switch_embedding_mode(tool, mode: str):
        """
        Switch embedding model between 'query' and 'passage' modes.
        Use 'passage' for indexing, 'query' for searching.
        
        Required for self-hosted NIMs using OpenAI provider.
        """
        from chromadb.utils.embedding_functions import OpenAIEmbeddingFunction
        
        if mode not in ["query", "passage"]:
            raise ValueError(f"Invalid mode '{mode}'. Must be 'query' or 'passage'.")
        
        embedding_function = OpenAIEmbeddingFunction(
            api_key=os.environ.get("NVIDIA_EMBED_TOKEN"),
            api_base=os.environ.get("NVIDIA_EMBED_BASE_URL"),
            model_name=f"nvidia/nv-embedqa-e5-v5-{mode}"
        )
        
        tool.adapter.embedchain_app.db.collection._embedding_function = embedding_function
        print(f"✓ Switched to '{mode}' mode (model: nvidia/nv-embedqa-e5-v5-{mode})")
```

**Usage:**

```python
# Create agent with WebsiteSearchTool (uses passage mode during initialization)
overview_writer = Agent(
    config=agents_config['overview_writer'], 
    tools=[website_search_tool]
)

# Switch to query mode after indexing completes
if OPENSHIFT_MODE:
    for tool in overview_writer.tools:
        if tool.name == 'Search in a specific website':
            switch_embedding_mode(tool, mode="query")
            break
```

**Why this is needed:**
- NVIDIA API Catalog handles mode switching automatically
- Self-hosted NIMs require explicit mode specification
- Indexing with wrong mode produces poor search results

## Environment Variables and Config

### RHOAI Workbench Configuration

Set these environment variables when creating the workbench:

| Variable | Value | Purpose |
|----------|-------|---------|
| `OPENSHIFT_MODE` | `true` | Enable RHOAI mode |
| `OPENAI_API_BASE` | `https://llama-project.apps.cluster.com/v1` | LLM endpoint |
| `OPENAI_API_KEY` | `your_llm_token` | LLM authentication |
| `NVIDIA_EMBED_BASE_URL` | `https://embed-project.apps.cluster.com/v1` | Embedding endpoint |
| `NVIDIA_EMBED_TOKEN` | `your_embed_token` | Embedding authentication |

Optional overrides:
- `LLM_PROVIDER`: Default is `openai`
- `LLM_MODEL`: Default is `meta/llama-3.1-8b-instruct`

### Deploying NIMs on RHOAI

1. **Deploy LLM NIM:**
   - Model: `meta/llama-3.1-8b-instruct` (or your choice)
   - Resources: 1 GPU minimum, 16Gi RAM, 4 CPU
   - Enable token authentication
   - Copy inference endpoint URL (e.g., `https://llama-project.apps.cluster.com/v1`)
   - Copy token from secret

2. **Deploy Embedding NIM:**
   - Model: `nvidia/nv-embedqa-e5-v5`
   - Resources: 1 GPU minimum, 16Gi RAM, 4 CPU
   - Enable token authentication
   - Copy inference endpoint URL (e.g., `https://embed-project.apps.cluster.com/v1`)
   - Copy token from secret

## Known Issues and Gotchas

### Issue: "Model not found" error with self-hosted NIMs

**Cause:** Model name in code doesn't match deployed NIM model ID.

**Solution:** 
1. Check deployed NIM's model ID in RHOAI UI
2. Update agent config YAML or `LLM_MODEL` environment variable
3. Keep `openai/` provider prefix, only change model name

Example: If you deployed `meta/llama-3.3-70b-instruct`:
```python
llm = LLM(
    model="openai/meta/llama-3.3-70b-instruct",  # Match your deployed model
    base_url=os.getenv('OPENAI_API_BASE'),
    api_key=os.getenv('OPENAI_API_KEY')
)
```

### Issue: Poor embedding search quality

**Cause:** Using wrong embedding mode (passage vs. query).

**Solution:** Ensure mode switching happens after agent creation:
1. Tool initialized with `passage` mode during indexing
2. Call `switch_embedding_mode(tool, "query")` before using the agent
3. Only required in RHOAI mode

### Issue: "Connection refused" or "401 Unauthorized"

**Cause:** Incorrect endpoint URL or token.

**Solution:**
1. Verify NIM is running (check RHOAI dashboard status)
2. Verify endpoint URL ends with `/v1`
3. Verify token is correct (regenerate if needed)
4. Test with curl:
   ```bash
   curl -X POST "https://llama-project.apps.cluster.com/v1/chat/completions" \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"model": "meta/llama-3.1-8b-instruct", "messages": [{"role": "user", "content": "Hello!"}]}'
   ```

### Issue: Agent config override not working

**Cause:** Override function called after agent creation instead of before.

**Solution:** Call `override_agent_llm_config()` immediately after loading YAML, before creating agents:

```python
# ✓ Correct order
with open('config/agents.yaml', 'r') as f:
    agents_config = yaml.safe_load(f)

agents_config = override_agent_llm_config(agents_config)  # Override FIRST

agent = Agent(config=agents_config['agent_name'])  # Create agent SECOND
```

## Dependencies

CrewAI blueprints on RHOAI require:
- `crewai` - Core framework
- `crewai-tools` - Tool implementations (WebsiteSearchTool, etc.)
- `chromadb` - Vector storage for embeddings
- `pysqlite3-binary` - SQLite compatibility (see [chromadb-on-rhoai.md](chromadb-on-rhoai.md))

## Testing Notes

Verify RHOAI mode works correctly:

1. **Environment detection:**
   ```python
   import os
   print(f"OPENSHIFT_MODE: {os.getenv('OPENSHIFT_MODE')}")
   print(f"Expected: true")
   ```

2. **LLM connectivity:**
   ```python
   from crewai import LLM
   
   llm = LLM(
       model="openai/meta/llama-3.1-8b-instruct",
       base_url=os.getenv('OPENAI_API_BASE'),
       api_key=os.getenv('OPENAI_API_KEY')
   )
   
   response = llm.call(messages=[{"role": "user", "content": "Hello!"}])
   print(response)  # Should get a response from self-hosted NIM
   ```

3. **Embedding mode:**
   - Verify tool initializes with `passage` mode
   - Verify switch to `query` mode succeeds
   - Verify search results are relevant

## Related Patterns

- [chromadb-on-rhoai.md](chromadb-on-rhoai.md) - SQLite compatibility patch
- [deployment-types/rhoai-notebook-pattern.md](../deployment-types/rhoai-notebook-pattern.md) - Notebook deployment
