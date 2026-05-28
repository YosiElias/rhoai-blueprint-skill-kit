# NVIDIA Blueprint to RHOAI Converter

Claude Code skills for converting NVIDIA AI Blueprints to Red Hat OpenShift AI (RHOAI) compatible deployments.

## Overview

This repository provides two complementary skills:

1. **Knowledge Extraction** (`bp-extract-blueprint-knowledge`) - Extract reusable patterns from completed RHOAI conversions
2. **Blueprint Conversion** (`bp-convert-to-rhoai`) - Apply proven patterns to convert new blueprints to RHOAI

## How It Works

### The Knowledge Base Approach

Instead of hardcoding conversion logic, these skills use a **knowledge base** built from your team's actual RHOAI conversions:

1. **Extract patterns** from completed conversions using `/bp-extract-blueprint-knowledge`
2. **Build knowledge base** with atomic, reusable component patterns
3. **Apply patterns** automatically when converting new blueprints with `/bp-convert-to-rhoai`
4. **Learn continuously** - each new conversion adds to the knowledge base

### Key Features

- **80/20 approach**: 80% pattern matching (automatic), 20% custom decisions (with user input)
- **Minimal changes**: Adds RHOAI support via `OPENSHIFT_MODE` conditionals, preserves original functionality
- **Smart retrieval**: Only loads relevant knowledge based on blueprint components
- **Red Hat best practices**: Queries official OpenShift docs when needed (via Context7)
- **User decisions**: Asks about model deployment strategy (local vs hosted API), GPU allocation, etc.

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- Access to completed RHOAI blueprint repositories (for knowledge extraction)
- OpenShift cluster 4.x+ (for testing conversions)
- `oc`, `helm`, `yq`, `git` CLI tools installed

## Quick Start

### Step 1: Build Knowledge Base

Extract patterns from your team's completed RHOAI conversions:

```bash
# Clone this repository
cd claude-skill-for-nvidia-blueprint-to-helm

# Run Claude Code
claude

# Extract knowledge from a completed RHOAI blueprint
/bp-extract-blueprint-knowledge --source https://github.com/NVIDIA-AI-Blueprints/nv-ingest-rag --fork https://github.com/your-org/nv-ingest-rag-rhoai
```

The skill will:
- Clone the repository
- Analyze RHOAI-specific commits
- Extract component patterns (Redis, Triton, Milvus, etc.)
- Generate knowledge files in `bp-convert-to-rhoai/knowledge-base/`

**Review and refine** the generated knowledge files before using them.

Repeat for all your completed RHOAI blueprints to build a comprehensive knowledge base.

### Step 2: Convert a New Blueprint

Apply proven patterns to convert a new NVIDIA Blueprint:

```bash
# Download the new NVIDIA Blueprint
git clone https://github.com/NVIDIA-AI-Blueprints/some-blueprint
cd some-blueprint

# Start Claude Code
claude

# Convert the blueprint
/bp-convert-to-rhoai .
```

The skill will:
1. Analyze the blueprint structure
2. Retrieve relevant knowledge patterns
3. Ask you about model deployment strategy and other decisions
4. Generate minimal RHOAI modifications
5. Create test plan and documentation

### Step 3: Test the Conversion

Follow the generated `TEST-PLAN.md`:

```bash
# Deploy to OpenShift with RHOAI mode enabled
helm install my-blueprint ./ \
  --set openshiftMode=true \
  --set serverHost=my-app.apps.cluster.example.com \
  --namespace my-namespace \
  --create-namespace

# Verify deployment
oc get pods -n my-namespace
```

## What These Skills Do

### Knowledge Extraction Skill

Analyzes completed RHOAI blueprint conversions and extracts:

**Component Patterns:**
- How components (Redis, Triton, Milvus, etc.) were adapted for RHOAI
- Security contexts, storage, GPU allocation
- `OPENSHIFT_MODE` conditional logic

**Architecture Patterns:**
- RAG pipelines, agentic workflows
- Inter-service communication
- Initialization order

**Deployment Patterns:**
- Helm chart structures with conditionals
- Notebook adaptations
- `oc apply` manifest organization

**Resource Patterns:**
- GPU allocation (node selectors, tolerations)
- Storage (PVC patterns, access modes)
- Networking (Routes, service mesh)
- Security (SCC requirements)

### Blueprint Conversion Skill

Converts new NVIDIA Blueprints to RHOAI-compatible versions:

**Analysis Phase:**
- Identifies components, architecture, deployment method
- Extracts resource requirements (GPU, storage, networking)

**Retrieval Phase:**
- Scores knowledge files by relevance (tag-based matching)
- Loads only relevant patterns (no context overflow)
- Retrieves additional knowledge on-demand

**Reasoning Phase:**
- Applies proven patterns from knowledge base
- Asks user about deployment strategy (local models vs NVIDIA API)
- Queries Red Hat docs (Context7) for gaps
- Checks guardrails (GPU, storage, security, networking, etc.)

**Generation Phase:**
- Minimal invasive changes with `OPENSHIFT_MODE` conditionals
- Prefers editing existing files over creating new ones
- Right tool for the job (Helm for multi-service, oc apply for notebooks)
- Generates test plan and documentation

## Directory Structure

```
claude-skill-for-nvidia-blueprint-to-helm/
├── README.md (this file)
├── .claude/
│   └── skills/
│       ├── bp-extract-blueprint-knowledge/
│       │   └── SKILL.md                    # Knowledge extraction skill
│       └── bp-convert-to-rhoai/
│           ├── SKILL.md                    # Blueprint conversion skill
│           ├── reasoning-guardrails.md     # Concern areas to check
│           ├── output-templates.md         # Output templates
│           ├── subagent-validation-prompt.md  # Validation instructions
│           └── knowledge-base/             # Extracted patterns
│               ├── README.md               # Knowledge base guide
│               ├── components/             # Component-specific patterns
│               ├── architectures/          # Architecture patterns
│               ├── deployment-types/       # Deployment method patterns
│               ├── resource-patterns/      # Cross-cutting concerns
│               └── integrations/           # Multi-component integrations
└── tmp/                                    # Temporary working directory
```

## Knowledge Base Structure

The knowledge base organizes patterns into atomic, reusable files:

**Component Patterns** (`components/`):
- `redis-on-rhoai.md` - How to deploy Redis on RHOAI
- `triton-on-rhoai.md` - Triton Inference Server patterns
- `milvus-on-rhoai.md` - Milvus vector database patterns
- One file per component, may contain multiple approaches

**Architecture Patterns** (`architectures/`):
- `rag-pipeline-pattern.md` - RAG architecture on RHOAI
- `agentic-workflow-pattern.md` - Agentic systems patterns

**Deployment Patterns** (`deployment-types/`):
- `helm-conditional-support.md` - OPENSHIFT_MODE in Helm
- `notebook-adaptation.md` - Jupyter notebook deployment

**Resource Patterns** (`resource-patterns/`):
- `gpu-allocation-openshift.md` - GPU scheduling, tolerations
- `storage-pvc-patterns.md` - PVC configurations, access modes
- `networking-routes-ingress.md` - OpenShift Routes
- `security-contexts-scc.md` - Security Context Constraints

## Usage Examples

### Example 1: Building Knowledge Base

```bash
# You have 3 completed RHOAI blueprints
claude

# Extract from first blueprint
/bp-extract-blueprint-knowledge --source https://github.com/NVIDIA-AI-Blueprints/nv-ingest-rag --fork https://github.com/your-org/nv-ingest-rag-rhoai

# Review generated knowledge files
cat .claude/skills/bp-convert-to-rhoai/knowledge-base/components/redis-on-rhoai.md

# Extract from second blueprint (adds to or updates existing knowledge)
/bp-extract-blueprint-knowledge --source https://github.com/NVIDIA-AI-Blueprints/video-search --fork https://github.com/your-org/video-search-rhoai

# Review updated/new knowledge
# Notice: if both used Redis, redis-on-rhoai.md now has "Approach A" and "Approach B"

# Extract from third blueprint
/bp-extract-blueprint-knowledge --source https://github.com/NVIDIA-AI-Blueprints/agent-studio --fork https://github.com/your-org/agent-studio-rhoai
```

### Example 2: Converting RAG Pipeline Blueprint

```bash
# Download new blueprint
git clone https://github.com/NVIDIA-AI-Blueprints/rag-chatbot
cd rag-chatbot

claude

# Start conversion
/bp-convert-to-rhoai .
```

**Conversion flow:**
1. Skill analyzes: docker-compose with Triton, Milvus, Redis, Llama-3.1-70b
2. Retrieves knowledge: `triton-on-rhoai.md`, `milvus-on-rhoai.md`, `redis-on-rhoai.md`, `triton-milvus-integration.md`, `gpu-allocation-openshift.md`
3. Asks user: "Llama-3.1-70b requires 2x A100. Deploy locally or use NVIDIA API?"
4. User chooses: "Local deployment"
5. Generates: Helm chart with `openshiftMode` conditionals, GPU allocation, PVCs, Routes
6. Creates: `TEST-PLAN.md`, `RHOAI-CONVERSION.md`, updated README

### Example 3: Converting Simple Notebook

```bash
git clone https://github.com/NVIDIA-AI-Blueprints/jupyter-inference
cd jupyter-inference

claude
/bp-convert-to-rhoai .
```

**Conversion flow:**
1. Analyzes: Single Jupyter notebook with Triton connection
2. Retrieves: `notebook-adaptation.md`, `triton-on-rhoai.md` (client-side)
3. Generates: `oc apply` manifests (no Helm - it's just a notebook)
4. Creates: `notebook-deployment.yaml`, `route.yaml`, `TEST-PLAN.md`

## Conversion Output

The conversion skill generates:

**Modified Files:**
- Existing blueprint files with `OPENSHIFT_MODE` conditional support
- values.yaml: Added `openshiftMode` flag (default: false)
- templates: Added conditional RHOAI configuration

**New Files (when necessary):**
- `templates/routes.yaml`: OpenShift Routes (no docker-compose equivalent)
- `TEST-PLAN.md`: Comprehensive testing guide
- `RHOAI-CONVERSION.md`: What changed and why
- Updated `README.md`: RHOAI deployment instructions

**For Helm deployments:**
```yaml
# values.yaml
openshiftMode: false  # Original behavior by default

# When deploying to RHOAI
helm install my-app ./ --set openshiftMode=true
```

**For oc apply deployments:**
```yaml
# Environment variable approach
OPENSHIFT_MODE=true oc apply -f manifests/
```

## Knowledge File Format

Each knowledge file uses YAML frontmatter for retrieval:

```markdown
---
type: component
components: [redis]
deployment_types: [helm, docker-compose]
resource_types: [storage, security-context]
architecture: []
source_examples:
  - blueprint: "nv-ingest-rag"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/nv-ingest-rag"
    fork_repo: "https://github.com/your-org/nv-ingest-rag-rhoai"
    notes: "Redis with PVC and anyuid SCC"
    approach: "A"
---

# Redis on RHOAI

## Overview
[Description]

## Conversion Pattern
[How to add RHOAI support]

### Security Context
\```yaml
securityContext:
  runAsUser: 999
  fsGroup: 999
\```

### Storage
\```yaml
volumeClaim Templates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
\```

## Known Issues
- [Issue and solution]
```

## How Retrieval Works

The conversion skill uses tag-based matching to find relevant knowledge:

1. **Extract blueprint features**: components, architecture, deployment types, resource needs
2. **Score knowledge files**: Component match = 10 points, Architecture = 5, Deployment = 4, Resource = 2
3. **Load top-scored files**: Only top 10 relevant files loaded (avoids context overflow)
4. **On-demand retrieval**: Load additional knowledge as specific questions arise

**Example scoring:**

| Knowledge File | Score Breakdown | Total |
|---|---|---|
| triton-milvus-integration.md | 20 (both components) + 5 (RAG arch) + 2 (networking) | **27** |
| triton-on-rhoai.md | 10 (triton) + 5 (RAG) + 4 (docker-compose) + 4 (gpu+storage) | **23** |
| milvus-on-rhoai.md | 10 (milvus) + 5 (RAG) + 4 (docker-compose) + 4 (storage+net) | **23** |
| redis-on-rhoai.md | 10 (redis) + 4 (docker-compose) + 2 (storage) | **16** |
| postgresql-on-rhoai.md | 0 (no matches) | **0** (excluded) |

## Conversion Philosophy

### OPENSHIFT_MODE Conditionals

Modifications are conditional - blueprint works on both platforms:

```yaml
# Helm template
{{- if .Values.openshiftMode }}
# RHOAI-specific: GPU node selectors, tolerations, anyuid SCC
securityContext:
  runAsUser: 0
nodeSelector:
  nvidia.com/gpu.present: "true"
{{- else }}
# Original behavior
securityContext: {}
{{- end }}
```

### Minimal Invasive Changes

- Prefer modifying existing files over creating new ones
- Add conditionals rather than replacing entire sections
- Only create new files when no equivalent exists (e.g., OpenShift Routes)

### Right Tool for the Job

- **Simple notebooks** → `oc apply` manifests
- **Multi-service apps** → Helm charts (all resources Helm-managed)
- **docker-compose** → Helm charts with conditional logic

## User Decision Points

The conversion skill asks for user input on:

1. **Model deployment strategy**:
   - Local deployment (GPU-intensive, low latency)
   - NVIDIA hosted API (no GPU needed, pay-per-use)
   - Hybrid (mix of local and API)

2. **GPU allocation approach**:
   - Dedicated GPU node pools (guaranteed access)
   - Shared GPU pools (flexible scheduling)

3. **Multiple valid patterns**:
   - When knowledge base shows different approaches for same component
   - User chooses based on their cluster setup

## Troubleshooting

### Knowledge Extraction Issues

**Problem**: Extraction skill generates incomplete knowledge files
**Solution**: 
- Manually review and refine generated files
- Check if RHOAI commits have clear messages
- Add missing patterns based on team knowledge

**Problem**: Can't find RHOAI-specific changes in git history
**Solution**:
- Check if commits mention "openshift", "rhoai", or "OPENSHIFT_MODE"
- Look for conditional blocks in Helm templates
- Analyze security context, PVC, and Route changes

### Conversion Issues

**Problem**: Conversion skill doesn't retrieve relevant knowledge
**Solution**:
- Check knowledge file frontmatter tags are accurate
- Verify blueprint components extracted correctly
- Manually specify knowledge files if automatic retrieval fails

**Problem**: Generated conversion doesn't work on OpenShift
**Solution**:
- Review TEST-PLAN.md and follow verification steps
- Check pod logs: `oc logs <pod-name>`
- Verify SCC bindings: `oc get pod <pod> -o yaml | grep scc`
- Check GPU allocation: `oc describe pod <pod> | grep nvidia.com/gpu`

## Contributing to Knowledge Base

When you complete a new RHOAI conversion:

1. Run extraction skill on your new conversion
2. Review and refine generated/updated knowledge files
3. Test knowledge by using it to convert a similar blueprint
4. Commit knowledge changes with clear messages
5. Share learnings with the team

## Advanced Features

### Context7 Integration

When knowledge gaps exist, the skill queries Red Hat's official documentation:

```python
# Example: Unknown technology "FUSE mounts"
Query Context7: "OpenShift FUSE mount security context constraints"
→ Gets authoritative answer from Red Hat docs
→ Applies pattern to conversion
```

### GitHub Source Retrieval

When knowledge file lacks detail, fetch from source:

```python
# Knowledge file mentions pattern but not exact YAML
→ Check frontmatter source_examples
→ Clone referenced repository
→ Read actual implementation
→ Apply similar pattern to new blueprint
```

### Dynamic Reasoning with Guardrails

The skill thinks freely but ensures coverage of critical concerns:
- Resource allocation
- Security contexts
- Networking
- Storage
- Inter-service dependencies
- And 5 more (see `reasoning-guardrails.md`)

## Future Enhancements

Potential V2 additions:

- Automated deployment testing to OpenShift test cluster
- Unit test generation (Helm unittest files)
- Conversion quality scoring
- Knowledge base auto-update from new conversions
- Multi-blueprint pattern analysis
- Interactive tutorial mode

## Support

- Check `reasoning-guardrails.md` for conversion concern areas
- See `retrieval-algorithm.md` for knowledge scoring details
- Review `knowledge-base/README.md` for knowledge management
- Open issues for bugs or feature requests

## License

Apache 2.0

## Acknowledgments

- NVIDIA for publishing AI Blueprints
- Red Hat OpenShift AI team
- Your team's RHOAI conversion experience

---

**Version**: 1.0  
**Last Updated**: 2026-05-20
