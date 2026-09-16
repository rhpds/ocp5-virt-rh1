# Module Outlines

Phase 4 of the intake flow. Generate detailed outlines for each module.

## Check for Existing Outlines

Check `publishing-house/spec/modules/` for existing module outline files.

**If outlines already exist** (non-empty `.md` files matching `module-*.md`):
> "I see module outlines already in the repo: [list filenames]. Want me to validate
> them against the design, or are these ready to go?"

- If author says they're ready → validate section structure, proceed
- If author wants validation → check each outline against the spec guidelines

## Generate Outlines (Fresh Intake)

Module outlines MUST be generated from the written design.md — not from conversation
context. Read the design doc and the module outline template from the project repo.

Read the module outline template at the project's `publishing-house/spec/module-outline-template.md`.

For each module in the Module Map table in design.md, generate one outline file:
- Output directory: `publishing-house/spec/modules/`
- Naming: `module-01-<short-title>.md`, `module-02-<short-title>.md`, etc.
- Follow the template structure exactly
- Reflect what design.md says — do not invent content not in the spec

Use the Agent tool to spawn a fresh subagent for generation to avoid context bleed:

```
Read the design spec at <project_root>/publishing-house/spec/design.md.
Read the module outline template at <project_root>/publishing-house/spec/module-outline-template.md.

For each module in the Module Map table, generate one outline file:
- Output directory: <project_root>/publishing-house/spec/modules/
- Naming: module-01-<short-title>.md, module-02-<short-title>.md, etc.
- Follow the template structure exactly.
- Reflect what design.md says — do not invent content not in the spec.
```

## Re-intake After Rejection

When called after the rejection handler (00-rejection-handler.md), module outlines
already exist. Do NOT regenerate from scratch.

1. Read the updated design.md to identify what changed
2. Read the existing module outlines in `publishing-house/spec/modules/`
3. For each affected module, update only the sections that changed:
   - If learning objectives changed → update the objectives section in that module
   - If a module was added → generate a new outline file for it
   - If a module was removed → delete the outline file
   - If scope/content changed for a module → update the relevant sections
4. Leave untouched modules as-is

## Auto-Generate Design and Module Summaries

After outlines are written or updated (both fresh and re-intake paths), generate summaries
for spec.yaml so reviewers see a narrative overview without reading full markdown files.

1. Read `publishing-house/spec/design.md`
2. Read all module files from `publishing-house/spec/modules/`
3. Generate a 2-3 sentence `design_overview` summarizing the design's problem statement, target audience, and approach
4. For each module, generate a 1-2 sentence `overview` summarizing what the module covers and what the learner does
5. Write into `approval_checklist.content` in spec.yaml:

```yaml
approval_checklist:
  content:
    design_overview: "Generated summary here..."
    module_summaries:
      - title: "Module 1 Title"
        overview: "Generated 1-2 sentence overview..."
      - title: "Module 2 Title"
        overview: "Generated 1-2 sentence overview..."
```

## Initialize Module Statuses

After outlines are generated, update spec.yaml to add `status: not_started` to each module entry:

For each module in `spec.modules`:
- If no `status` field exists → add `status: not_started`
- If `status` already exists → leave it unchanged (handles re-intake)

Example spec.yaml module entry after this step:
```yaml
modules:
  - id: module-01
    title: "Pipeline Setup"
    duration_min: 30
    status: not_started
  - id: module-02
    title: "Configuring Triggers"
    duration_min: 25
    status: not_started
```

This ensures the development skill's writer can track sequential progress.

## Write Point

```bash
git add publishing-house/spec/modules/
git diff --cached --quiet || git commit -m "feat: phase 4 — module outlines generated" 2>/dev/null || true
```

Proceed to Phase 5 (infrastructure confirmation).
