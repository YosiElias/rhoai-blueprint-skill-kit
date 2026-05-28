# RHOAI Conversion Knowledge Base

This directory contains reusable patterns extracted from successfully completed NVIDIA Blueprint → RHOAI conversions.

## Purpose

The knowledge base enables the `bp-convert-to-rhoai` skill to:
- Apply proven conversion patterns to new blueprints
- Avoid reinventing solutions for common components
- Maintain consistency across conversions
- Learn from team's collective experience

## Directory Structure

```
knowledge-base/
├── components/          # Component-specific patterns (Redis, Triton, Milvus, etc.)
├── architectures/       # Architecture patterns (RAG pipeline, agentic, etc.)
├── deployment-types/    # Deployment patterns (Helm, notebooks, oc apply)
├── resource-patterns/   # Cross-cutting concerns (GPU, storage, networking, SCC)
└── integrations/        # Non-trivial multi-component integrations
```

## Knowledge File Format

Each knowledge file uses YAML frontmatter for tagging and retrieval:

```markdown
---
type: component | architecture | deployment-type | resource-pattern | integration
components: [redis, triton]           # Dimension A: which components
architecture: [rag-pipeline]          # Dimension B: architecture patterns
deployment_types: [helm, oc-apply]    # Dimension C: deployment methods
resource_types: [storage, gpu]        # Dimension D: RHOAI resource concerns
source_examples:
  - blueprint: "blueprint-name"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/..."
    fork_repo: "https://github.com/your-org/...-rhoai"
    notes: "Brief description"
    approach: "A"                     # Links to approach section
---

# Content
[Pattern details, code examples, gotchas]
```

## Adding Knowledge

### Method 1: Automated Extraction (Recommended)

Use the extraction skill on completed RHOAI blueprints:

```bash
/bp-extract-blueprint-knowledge https://github.com/your-org/blueprint-rhoai
```

The skill will:
- Analyze the repository
- Extract component patterns
- Generate/update knowledge files
- Create summary report

Review and refine the generated knowledge before using it.

### Method 2: Manual Addition

1. **Create or update a knowledge file** in the appropriate directory
2. **Use the standard frontmatter format** with accurate tags
3. **Include complete code examples** from actual conversions
4. **Document gotchas and edge cases**
5. **Link to source repositories** for reference

## Tagging Dimensions

Knowledge files are retrieved via tag matching. Use accurate tags:

### Dimension A: Components
Services/tools the pattern applies to:
- `redis`, `milvus`, `triton`, `postgresql`, `llama`, `mistral`, etc.

### Dimension B: Architecture
Architecture patterns:
- `rag-pipeline`, `agentic-workflow`, `inference-only`, `multi-model`

### Dimension C: Deployment Types
How services are deployed:
- `helm`, `docker-compose`, `oc-apply`, `notebook`

### Dimension D: Resource Types
RHOAI concerns addressed:
- `gpu`, `storage`, `networking`, `security-context`, `service-mesh`

## Handling Multiple Approaches

When a component can be deployed in different ways (e.g., Redis with different images):

1. **Add to the same file** (don't create separate files)
2. **Label each approach clearly**: "Approach A", "Approach B"
3. **Update frontmatter** to list all source examples
4. **Add "Choosing Between Approaches"** section

Example:
```markdown
## Approach A: Redis with Bitnami Image (from blueprint-x)
[Details]

## Approach B: Redis with Official Image (from blueprint-y)
[Details]

## Choosing Between Approaches
[Guidance on selection]
```

## Knowledge Quality Guidelines

### DO:
- ✅ Include complete, working code examples
- ✅ Copy actual YAML from successful conversions
- ✅ Document known issues and their solutions
- ✅ Explain **why** a pattern is used, not just **what** it is
- ✅ Link to source repositories
- ✅ Update existing files when finding new approaches

### DON'T:
- ❌ Include pseudo-code or incomplete examples
- ❌ Mix multiple approaches without clear separation
- ❌ Omit error handling or edge cases
- ❌ Create duplicate knowledge files for same component
- ❌ Use incorrect or misleading tags

## File Naming Conventions

- **Components**: `<component-name>-on-rhoai.md` (e.g., `redis-on-rhoai.md`)
- **Architectures**: `<architecture-name>-pattern.md` (e.g., `rag-pipeline-pattern.md`)
- **Deployment types**: `<deployment-type>-pattern.md` (e.g., `helm-conditional-support.md`)
- **Resource patterns**: `<resource-type>-<topic>.md` (e.g., `gpu-allocation-openshift.md`)
- **Integrations**: `<service-a>-<service-b>-integration.md` (e.g., `triton-milvus-integration.md`)

Use lowercase with hyphens, no underscores or spaces.

## Maintenance

### Regular Review
- Validate patterns still apply to new OpenShift/RHOAI versions
- Update code examples if APIs change
- Remove deprecated patterns

### Version Control
- Commit knowledge changes with clear messages
- Reference which blueprint prompted the update
- Review team changes before merging

### Quality Control
- Test knowledge by using it in actual conversions
- Gather feedback from team members
- Refine unclear or incomplete sections

## Using Knowledge in Conversions

The `bp-convert-to-rhoai` skill automatically:
1. Analyzes the new blueprint
2. Scores knowledge files by relevance (tag matching)
3. Loads top-scored knowledge
4. Applies patterns during conversion

You don't manually select knowledge files - the retrieval algorithm handles it.

## Contributing

When you encounter a new pattern or improve an existing one:

1. Run extraction skill on the blueprint
2. Review generated/updated knowledge files
3. Refine for clarity and completeness
4. Commit to version control
5. Share learnings with the team

## Questions or Issues

If knowledge extraction seems incorrect or incomplete:
- Check the source blueprint repository for clarity
- Manually review and correct the knowledge file
- Document what was unclear in extraction notes
- Consider improving the extraction skill logic

---

**Last Updated**: 2026-05-20
**Knowledge Base Version**: 1.0
