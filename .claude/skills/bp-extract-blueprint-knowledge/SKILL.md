---
name: bp-extract-blueprint-knowledge
description: Extract RHOAI conversion patterns from a completed blueprint and generate knowledge base files
argument-hint: --source <original-nvidia-blueprint-url> --fork <rhoai-fork-url>
allowed-tools: Bash, Read, Write, Edit, WebFetch, Agent
---

# Blueprint Knowledge Extraction Skill

You are extracting reusable RHOAI conversion patterns from an already-completed NVIDIA Blueprint that has been adapted for Red Hat OpenShift AI (RHOAI).

## Goal

Generate atomic, reusable knowledge files that capture how components were adapted for RHOAI. These knowledge files will be used by the `bp-convert-to-rhoai` skill to automatically apply proven patterns to new blueprints.

## Input

The user provides two GitHub repository URLs:
- `--source <url>`: Original NVIDIA Blueprint repository
- `--fork <url>`: RHOAI-adapted fork of the blueprint

## Workflow

### Step 1: Clone Fork Repository

Clone the RHOAI fork repository:

```bash
# Parse arguments
SOURCE_URL=""
FORK_URL=""
while [[ $# -gt 0 ]]; do
    case $1 in
        --source) SOURCE_URL="$2"; shift 2 ;;
        --fork) FORK_URL="$2"; shift 2 ;;
        *) echo "Unknown argument: $1"; exit 1 ;;
    esac
done

if [ -z "$SOURCE_URL" ] || [ -z "$FORK_URL" ]; then
    echo "Usage: --source <url> --fork <url>"
    exit 1
fi

# Set up working directory
WORK_DIR="/tmp/blueprint-extraction-$$"
mkdir -p "$WORK_DIR"
cd "$WORK_DIR"

# Clone fork (RHOAI-adapted version)
FORK_NAME=$(basename "$FORK_URL" .git)
git clone "$FORK_URL" "$FORK_NAME"
cd "$FORK_NAME"
```

### Step 2: Identify Blueprint Type and Components

Determine what type of blueprint this is:

```bash
# Find deployment artifacts
find . -name "docker-compose.yaml" -o -name "docker-compose.yml" -o -name "Chart.yaml" -o -name "*.ipynb" | head -10

# List root directory
ls -la

# Check for common structure patterns
test -f docker-compose.yaml && echo "HAS_DOCKER_COMPOSE"
test -f Chart.yaml && echo "HAS_HELM"
find . -name "*.ipynb" -type f | head -1 && echo "HAS_NOTEBOOKS"
```

**Questions to answer:**
- Primary deployment method: Helm charts, docker-compose, notebooks, or oc apply manifests?
- What components/services are deployed?
- What architecture pattern: RAG pipeline, agentic workflow, inference-only?
- What resources are needed: GPU, storage, networking?

### Step 3: Identify Fork Point and RHOAI Changes

Add source repository as upstream remote and fetch it:

```bash
# Add source repo as upstream remote
git remote add upstream "$SOURCE_URL"

# Fetch source commits (downloads source into fork's git database)
git fetch upstream

# Find fork point (common ancestor between fork and source)
UPSTREAM_MAIN=$(git ls-remote --symref upstream HEAD | grep "ref:" | awk '{print $2}' | sed 's|refs/heads/||')
if [ -z "$UPSTREAM_MAIN" ]; then
    UPSTREAM_MAIN="main"  # fallback to main if detection fails
fi

# Find the merge-base (where fork originally diverged from source)
FORK_POINT=$(git merge-base HEAD upstream/$UPSTREAM_MAIN 2>/dev/null || git merge-base HEAD upstream/master 2>/dev/null)

if [ -n "$FORK_POINT" ]; then
    echo "Fork point: $FORK_POINT"
    
    # Get commits added after fork (RHOAI changes only, not new upstream commits)
    git log --oneline "$FORK_POINT..HEAD"
    
    # Show files changed in RHOAI fork
    git diff --name-only "$FORK_POINT..HEAD"
else
    echo "Could not find fork point - analyzing recent commits"
    git log --oneline -30
fi
```

**Important:** `git merge-base` finds the common ancestor where the fork originally diverged. This means:
- New commits in the source that haven't been merged into the fork are NOT included
- Only commits unique to the fork (RHOAI changes) are identified
- Even if the fork is outdated, we only extract RHOAI-specific patterns

### Step 4: Analyze RHOAI-Specific Commits

Use the Agent tool to intelligently filter commits and identify RHOAI-related changes:

```bash
# Get all commits added after fork point
if [ -n "$FORK_POINT" ]; then
    # Show commits added after fork
    git log --oneline "$FORK_POINT..HEAD" > commits-after-fork.txt
    
    # Show detailed diff for RHOAI changes
    git diff "$FORK_POINT..HEAD" -- "*.yaml" "*.yml" > yaml-changes.diff
    git diff "$FORK_POINT..HEAD" -- "*.md" "*.sh" > other-changes.diff
    
    # Show files changed
    git diff --name-status "$FORK_POINT..HEAD" > changed-files.txt
else
    echo "No fork point found - will analyze all commits"
    git log --oneline -30 > commits-after-fork.txt
    git diff --name-status HEAD~30..HEAD > changed-files.txt
fi
```

**Agent Analysis**: Use Agent tool to analyze commits and filter RHOAI-related changes:
- Read commits-after-fork.txt and changed-files.txt
- Identify which commits are RHOAI-related vs unrelated (typos, general bugs, etc.)
- Focus on commits that modify:
  - OPENSHIFT_MODE environment variable checks
  - Helm templates with conditional blocks ({{- if .Values.openshiftMode }})
  - Security contexts (SCC, runAsUser, fsGroup)
  - PVC/storage configurations
  - Route/Ingress definitions
  - GPU resource specifications
- Filter out commits that are clearly non-RHOAI (typo fixes, README formatting, etc.)

**Key indicators of RHOAI modifications:**
- Files with `OPENSHIFT_MODE` environment variable checks
- Helm templates with conditional blocks
- Security context additions (SCC, runAsUser, fsGroup)
- PVC/storage configurations
- Route/Ingress definitions
- GPU resource specifications

### Step 5: Extract Components

Identify all components/services:

**From docker-compose.yaml:**
```bash
if [ -f docker-compose.yaml ]; then
    yq '.services | keys' docker-compose.yaml
elif [ -f docker-compose.yml ]; then
    yq '.services | keys' docker-compose.yml
fi
```

**From Helm charts:**
```bash
if [ -f Chart.yaml ]; then
    # Check values.yaml for service definitions
    yq '.images | keys' values.yaml 2>/dev/null || echo "No images key"
    # List service templates
    find templates/services -name "*.yaml" 2>/dev/null | xargs -I{} basename {} .yaml
fi
```

**Common components:**
- Inference servers: Triton, vLLM, TensorRT-LLM
- NIM models: llama, mistral, embed-qa, nemo-retriever (note if deployable locally or API-only)
- Vector DBs: Milvus, Qdrant, Weaviate, pgvector
- Caches: Redis, Memcached
- Databases: PostgreSQL, MySQL, MongoDB
- Message queues: RabbitMQ, Kafka
- Storage: MinIO, S3
- Web servers: nginx, Apache
- Other: Jaeger, Prometheus, etc.

### Step 6: Analyze Component-Level Patterns

For each component, extract:

#### 5.1 OPENSHIFT_MODE Conditional Logic

```bash
# Find OPENSHIFT_MODE usage
grep -r "OPENSHIFT_MODE" . --include="*.yaml" --include="*.yml" --include="*.tpl"
```

How is conditional support implemented?
- Helm template conditionals: `{{- if .Values.openshiftMode }}`
- Environment variable checks in code
- Separate config files for RHOAI

#### 5.2 Security Context Patterns

```bash
grep -r "securityContext" . --include="*.yaml" --include="*.yml" -A 5
grep -r "runAsUser\|fsGroup\|runAsNonRoot" . --include="*.yaml"
grep -r "anyuid\|privileged" . --include="*.yaml" --include="*.sh"
```

#### 5.3 Storage Patterns

```bash
grep -r "PersistentVolumeClaim\|kind: PVC" . --include="*.yaml"
grep -r "storageClass" . --include="*.yaml"
grep -r "ReadWriteOnce\|ReadWriteMany" . --include="*.yaml"
```

#### 5.4 GPU Allocation

```bash
grep -r "nvidia.com/gpu" . --include="*.yaml"
grep -r "tolerations" . --include="*.yaml" -A 3 | grep -i "gpu\|nvidia"
grep -r "nodeSelector" . --include="*.yaml" -A 3 | grep -i "gpu\|nvidia"
```

#### 5.5 Networking (Routes, Services)

```bash
grep -r "kind: Route" . --include="*.yaml" -A 10
grep -r "kind: Ingress" . --include="*.yaml" -A 10
```

### Step 7: Generate Knowledge Files

**Target directory:** `../bp-convert-to-rhoai/knowledge-base/`

For each component, determine if knowledge file already exists:

```bash
KNOWLEDGE_BASE="../bp-convert-to-rhoai/knowledge-base"
COMPONENT_NAME="redis"  # example
KNOWLEDGE_FILE="$KNOWLEDGE_BASE/components/${COMPONENT_NAME}-on-rhoai.md"

if [ -f "$KNOWLEDGE_FILE" ]; then
    echo "Knowledge file exists - analyzing if this is a new approach or same pattern"
    # Read existing file to determine next steps
else
    echo "Creating new knowledge file"
fi
```

**When knowledge file exists, use Agent or dynamic reasoning to:**
1. **Compare patterns:** Read existing approaches in the file
2. **Identify differences:** Check for different:
   - Container images (official vs Bitnami vs custom)
   - Deployment methods (Deployment vs StatefulSet vs subchart)
   - Configuration (persistence vs emptyDir, auth vs no-auth)
   - Security contexts (anyuid SCC vs restricted)
3. **Make decision:**
   - **Same pattern:** Add blueprint to existing approach's `source_examples`
   - **Different pattern:** Create new "Approach X" section
4. **Document rationale:** In extraction summary, explain why approaches differ

#### 6.1 Knowledge File Structure

**If file DOES NOT exist**, create new file:

```markdown
---
type: component
components: [<component-name>]
deployment_types: [helm|docker-compose|oc-apply|notebook]
resource_types: [gpu|storage|networking|security-context]
architecture: []  # Fill if specific to certain architectures
source_examples:
  - blueprint: "<blueprint-name>"
    source_repo: "<original-nvidia-blueprint-url>"
    fork_repo: "<rhoai-fork-url>"
    notes: "<what pattern this demonstrates>"
    approach: "A"
---

# <Component Name> on RHOAI

## Overview
[Brief description of the component and its role]

## Conversion Pattern

### OPENSHIFT_MODE Conditional Support
[How conditional logic is implemented - show code examples]

### Deployment Type: [Helm|OC Apply|Notebook]
[How this component is deployed]

### Security Context Requirements
[SCC requirements, runAsUser, fsGroup, etc.]

**Example:**
```yaml
[security context YAML from repo]
```

### Storage Configuration
[If applicable - PVC patterns, storageClass, access modes]

**Example:**
```yaml
[PVC configuration from repo]
```

### GPU Resource Allocation
[If applicable - node selectors, tolerations, nvidia.com/gpu]

**Example:**
```yaml
[GPU configuration from repo]
```

### Networking Configuration
[Routes, Services, Ingress patterns]

**Example:**
```yaml
[networking config from repo]
```

### Environment Variables and Config
[RHOAI-specific env vars, ConfigMaps, Secrets]

**Example:**
```yaml
[env config from repo]
```

## Known Issues and Gotchas
[Common problems and solutions - extract from commit messages and comments]

- Issue: [description from commit message or comment]
  - Solution: [how it was resolved]

## Dependencies
[If this component requires other components]

## Testing Notes
[How to verify this component works on RHOAI]
```

**If file ALREADY EXISTS**, add new approach:

Read the existing file, then append:

```markdown
---

## Approach B: [Descriptive Name] (from <blueprint-name>)

### When to Use
[When this approach is preferred over Approach A]

### Differences from Approach A
[Key differences: image, configuration, deployment method]

### Conversion Pattern
[Same structure as above but for this specific approach]

---

## Choosing Between Approaches
[Guidance on selecting the right approach based on blueprint characteristics]
```

Update the frontmatter to include the new source example:

```yaml
source_examples:
  - blueprint: "existing-blueprint"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/..."
    fork_repo: "https://github.com/your-org/...-rhoai"
    notes: "..."
    approach: "A"
  - blueprint: "new-blueprint"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/..."
    fork_repo: "https://github.com/your-org/...-rhoai"
    notes: "..."
    approach: "B"
```

#### 6.2 Architecture Pattern Files

If the blueprint demonstrates a specific architecture (RAG, agentic, etc.), create:

`$KNOWLEDGE_BASE/architectures/<architecture-name>-pattern.md`

Include:
- Component orchestration
- Inter-service communication
- Data flow
- RHOAI-specific modifications
- Initialization order

#### 6.3 Deployment-Type Pattern Files

If notable deployment approach (e.g., Helm with sophisticated conditionals), create:

`$KNOWLEDGE_BASE/deployment-types/<deployment-type>-pattern.md`

Include:
- File organization
- OPENSHIFT_MODE conditional structure
- Common template patterns
- values.yaml organization

#### 6.4 Resource Pattern Files

For cross-cutting concerns, update or create:

- `$KNOWLEDGE_BASE/resource-patterns/gpu-allocation-openshift.md`
- `$KNOWLEDGE_BASE/resource-patterns/storage-pvc-patterns.md`
- `$KNOWLEDGE_BASE/resource-patterns/networking-routes-ingress.md`
- `$KNOWLEDGE_BASE/resource-patterns/security-contexts-scc.md`

Include patterns that apply across multiple components.

#### 6.5 Integration Patterns (Only When Non-Trivial)

**Only create if there's actual complexity**, such as:
- ✅ Triton + Milvus with specific init order and networking
- ✅ Service mesh configuration for multi-service communication
- ❌ Redis and PostgreSQL both running (trivial - just apply component patterns)

`$KNOWLEDGE_BASE/integrations/<service-a>-<service-b>-integration.md`

### Step 8: Generate Summary Report

Create a summary document showing what was extracted:

```markdown
# Knowledge Extraction Summary

**Blueprint**: <name>
**Repository**: <github-url>
**Extraction Date**: <date>
**Extraction Working Directory**: <work-dir>

## Blueprint Characteristics
- **Deployment Type**: [Helm|Docker-Compose|Notebooks|OC Apply]
- **Architecture**: [RAG|Agentic|Inference|Other]
- **GPU Required**: [Yes|No]
- **Services Count**: <number>

## Components Identified

### New Components
- **<component-name>**: Created new knowledge file at `components/<component-name>-on-rhoai.md`

### Updated Components
- **<component-name>**: Added Approach B to existing file `components/<component-name>-on-rhoai.md`

## Architecture Patterns
- **<architecture-name>**: Created/Updated `architectures/<architecture-name>-pattern.md`

## Deployment Patterns
- **<deployment-type>**: Created/Updated `deployment-types/<deployment-type>-pattern.md`

## Resource Patterns Updated
- GPU allocation patterns
- Storage/PVC patterns
- Networking patterns
- Security context patterns

## Integration Patterns
- **<integration-name>**: Created `integrations/<integration-name>-integration.md`

## Files Generated/Updated
[List of all knowledge files created or modified]

## Files Requiring Manual Review
[List files where extraction was uncertain or incomplete]

- **File**: `components/<name>.md`
  - **Reason**: [Why manual review is needed]
  - **Recommendation**: [What to check]

## Extraction Notes
[Any observations, patterns, or insights from this blueprint]

## Recommended Next Steps
1. Review generated knowledge files for accuracy
2. Validate code examples are complete and correct
3. Add missing edge cases or gotchas from team knowledge
4. Test knowledge files by using them to convert a similar blueprint
```

### Step 9: Output and Cleanup

1. **Save the summary report** to the working directory:
   ```bash
   # Save summary to working directory
   cat > "$WORK_DIR/EXTRACTION_SUMMARY_$(basename $FORK_URL .git).md" <<'EOF'
   [Summary content from Step 8]
   EOF
   ```

2. **Print the summary to console** for immediate review

3. **Print the working directory path** so the user can:
   - Review the cloned repository
   - Read the saved extraction summary
   - Inspect files that were analyzed

4. List all generated/modified knowledge files with their paths

5. Recommend the user review the draft knowledge files

**Do NOT delete the working directory automatically** - the user may want to inspect it.

The working directory ($WORK_DIR) contains:
- Cloned fork repository
- EXTRACTION_SUMMARY_<blueprint-name>.md file
- Any temporary analysis files

## Important Guidelines

### 1. Atomic Components
- One component per file (e.g., separate files for Redis, Milvus, Triton)
- Only combine components in integration files if there's non-trivial interaction

### 2. Accurate Tagging
- Ensure frontmatter tags accurately reflect what the knowledge applies to
- Use all relevant tags from dimensions A, B, C, D

### 3. Include Actual Code
- Copy relevant YAML/config snippets from the repository
- Show complete examples, not pseudo-code
- Preserve indentation and structure

### 4. Capture Gotchas
- Look for commit messages mentioning "fix", "issue", "problem"
- Extract solutions from code comments
- Document workarounds and their reasons

### 5. Link to Source
- Always include GitHub repo URL
- Reference specific files where patterns are found
- Note commit hashes if referencing specific fixes

### 6. Handling Multiple Approaches Intelligently

When extracting from a blueprint where a component knowledge file already exists, you must determine whether this represents a **new approach** or validates an **existing approach**.

**Core principle:** Only create separate approaches when the patterns are fundamentally incompatible. If someone following Approach A could reasonably adapt it to this blueprint with minor parameter changes, it's the same approach.

**Think like an engineer reviewing solutions:**
- Would you recommend these two implementations to different teams for different reasons? → Separate approaches
- Would you say "this is basically the same thing with different config values"? → Same approach, add to source examples

**What makes patterns fundamentally different:**
The deployment architecture, security model, or resource lifecycle differs in ways that require different implementation decisions. For example: choosing between a standalone pod and a StatefulSet affects how you handle storage, networking, and scaling. These are different approaches.

**What's superficial variation:**
Configuration parameters, resource quantities, naming conventions, minor version differences, or formatting preferences. These don't change how you'd approach the conversion - they're the same pattern with different settings.

**Your decision process:**
1. Read the existing approach(es) thoroughly
2. Extract the pattern from this blueprint  
3. Reason: "If I were converting a new blueprint with this component, would following the existing approach work, or would I need fundamentally different guidance?"
4. Document your reasoning in the extraction summary

**When in doubt:** Consolidate. It's easier to split approaches later than to merge duplicates. Multiple blueprints validating the same pattern increases confidence.

**Result:** Knowledge base stays lean and scales gracefully. The conversion skill gets clear choices between genuinely different strategies, not variations of the same thing.

### 7. Draft Quality
- Inform the user that extracted knowledge is draft quality
- Recommend team review for:
  - Accuracy of pattern extraction
  - Completeness of examples
  - Missing edge cases
  - Clarity of explanations

## Error Handling

If extraction fails or is uncertain:
- Document what was unclear in "Files Requiring Manual Review"
- Provide partial knowledge with clear notes about gaps
- Continue with other components rather than stopping completely

## Output Location

All knowledge files go to:
```
.claude/skills/bp-convert-to-rhoai/knowledge-base/
├── components/
├── architectures/
├── deployment-types/
├── resource-patterns/
└── integrations/
```

Use Write tool to create new files, Edit tool to update existing files.
