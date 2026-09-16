# Module Outlines from Content

Generate module outlines by reading the actual content modules and mapping them to the Publishing House format.

## Read Content Modules

1. Read the module outline template at `publishing-house/spec/module-outline-template.md`
2. Read `publishing-house/spec/design.md` — the Module Map table defines the module list
3. Read the corresponding content pages from `content/modules/ROOT/pages/` for each module
4. Read `content/modules/ROOT/nav.adoc` — current module references

## Generate Outlines and Rename Files

For each module in the Module Map table:

### 1. Generate outline file
- Output directory: `publishing-house/spec/modules/`
- Naming: `module-01-<short-title>.md`, `module-02-<short-title>.md`, etc.
- Follow the template structure exactly

### 2. Rename content file to match
- Old: `content/modules/ROOT/pages/old-name.adoc`
- New: `content/modules/ROOT/pages/module-01-<short-title>.adoc`
- The base name MUST match exactly (e.g., `module-01-pipeline-setup`)

### 3. Rename runtime-automation folder to match
- Old: `runtime-automation/old-name/`
- New: `runtime-automation/module-01-<short-title>/`
- Skip if the old folder doesn't exist
- The folder name MUST match the base name exactly

### 4. Update nav.adoc references
- Replace old filename references with new names
- Example: `* xref:old-name.adoc[Old Title]` → `* xref:module-01-pipeline-setup.adoc[Pipeline Setup]`

For each outline, derive the sections from the actual content page:

1. **Brief Overview** — summarize what the content page covers
2. **Audience and Time** — include the duration estimate from the Module Map
3. **Learning Objectives** — extract or derive from the content (use valid action verbs from policy)
4. **Lab Structure** — create a table of steps/sections from the content page's structure (headings, code blocks, instructions)
5. **Key Takeaways** — derive from the content's conclusion or the skills practiced

Use the Agent tool to spawn a fresh subagent for generation:

```
Read the design spec at <project_root>/publishing-house/spec/design.md.
Read the module outline template at <project_root>/publishing-house/spec/module-outline-template.md.
Read the content pages in <project_root>/content/modules/ROOT/pages/.

For each module in the Module Map table, generate one outline file:
- Output directory: <project_root>/publishing-house/spec/modules/
- Naming: module-01-<short-title>.md, module-02-<short-title>.md, etc.
- Follow the template structure exactly.
- Derive content from the corresponding page in content/modules/ROOT/pages/.
```

## Auto-Generate Design and Module Summaries

After outlines are written, generate summaries for spec.yaml:

1. Read `publishing-house/spec/design.md`
2. Read all module files from `publishing-house/spec/modules/`
3. Generate a 2-3 sentence `design_overview`
4. For each module, generate a 1-2 sentence `overview`
5. Write into `approval_checklist.content` in spec.yaml:

```yaml
approval_checklist:
  content:
    design_overview: "Generated summary here..."
    module_summaries:
      - title: "Module 1 Title"
        overview: "Generated 1-2 sentence overview..."
```

## Write Point

After all renames and outline generation:

```bash
git add publishing-house/spec/modules/ publishing-house/spec.yaml
git add content/modules/ROOT/pages/ content/modules/ROOT/nav.adoc
git add runtime-automation/
git diff --cached --quiet || git commit -m "feat: module outlines generated and content files renamed to match" 2>/dev/null || true
```

Proceed to Phase 5 (infrastructure confirmation).
