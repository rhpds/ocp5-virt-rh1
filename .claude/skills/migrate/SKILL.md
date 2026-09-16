---
name: rhdp-publishing-house:migrate
description: This skill handles migration intake for existing Showroom repos imported into the Publishing House. It reads the content/ folder, site.yml, and ui-config.yml to reverse-engineer spec.yaml, design.md, and module outlines, then submits for review.
---

---
context: main
model: claude-opus-4-6
---

# Migrate Orchestrator

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You handle migration intake by coordinating two agents: a **content reader** that analyzes the imported repo, and a **migration writer** that generates the Publishing House intake artifacts from that analysis.

## Tool Boundaries

**Do NOT use** Central API tools directly. All external interactions go through `publishing-house/tools/` scripts.

**Do NOT use** MCP tools. All external interactions go through `publishing-house/tools/` scripts.

See @rhdp-publishing-house/skills/common/user-interaction.md for how to present multi-option choices to the author.

## Steps 1–3 — Pre-flight

Follow @rhdp-publishing-house/skills/common/pre-flight.md (Steps 1–3: verify project, read identity, check auth).

### Step 4 — Workflow check

**RULE: This sequence runs every invocation. No exceptions. No skipping. No reusing previous results.**

**4a.** Get workflow data:
```bash
python publishing-house/tools/ph-workflow-data.py
```
If this fails → set `offline_mode = true`, skip to Step 5.
If this succeeds → extract `workflow_id`. Set `offline_mode = false`.

**4b.** Get workflow state (skip if offline):
```bash
python publishing-house/tools/ph-workflow-state.py WORKFLOW_ID
```
If stage is not `intake` → STOP. This is the only condition that stops the skill.
If offline → assume `intake`.

**4c.** Sync (skip if offline):
```bash
python publishing-house/tools/ph-sync.py
```
Extract `unresolved_rejections` from the output. Commit any changes:
```bash
git add publishing-house/spec.yaml catalog-info.yaml
git diff --cached --quiet || git commit -m "feat: sync workflow data from Central API" 2>/dev/null || true
```

**4d.** If `unresolved_rejections` > 0 → follow `@rhdp-publishing-house/skills/intake/procedures/00-rejection-handler.md`. After the rejection handler completes, continue with Step 5.

### Step 5 — Load policy and project files

1. Fetch validation policy:
   ```bash
   python publishing-house/tools/ph-policy.py
   ```
   If it fails and offline → check for `publishing-house/policy.json` as a static fallback.

2. Read `~/.config/publishing-house/policy.json` if it exists. Use these lists throughout:
   - `valid_content_types` — constrain content type choices
   - `valid_audiences` — constrain audience choices
   - `products` (with `aliases`) — validate product names
   - `action_verbs_valid` — learning objectives must start with one of these
   - `action_verbs_rejected` — reject objectives starting with these

3. Read `publishing-house/spec.yaml` — understand the structure, valid values from inline comments, and which fields are already populated

4. Read `publishing-house/spec/design.md` — understand the section structure from the placeholders

5. Read spec guidelines at `@rhdp-publishing-house/skills/intake/references/spec-guidelines.md`

6. Read the module outline template from the project repo at `publishing-house/spec/module-outline-template.md`

## Step 6 — Spawn Content Reader agent

Use the Agent tool to spawn a content reader agent. Pass the project root path so the agent knows where to find files.

```
Agent prompt:
You are a content reader for a Showroom lab migration. Read all the content files in the project and produce a structured analysis report.

Project root: <project_root>

Read these files:
1. <project_root>/site.yml — extract title, nav structure, metadata
2. <project_root>/ui-config.yml (if it exists) — extract UI config
3. <project_root>/content/modules/ROOT/nav.adoc — module order and titles
4. All .adoc files in <project_root>/content/modules/ROOT/pages/ — the actual lab content
5. <project_root>/publishing-house/spec.yaml — check pre-populated fields
6. <project_root>/catalog-info.yaml — check ph.rhdp.io/migrated-repo annotation

For each module page, extract:
- Title (from the page heading or nav.adoc reference)
- Section headings and structure
- Code blocks and commands used
- Products, operators, and tools mentioned
- Prerequisites assumed
- Approximate complexity and length

Produce a structured report with these sections:

## Title
The project title from site.yml

## Migrated Repo
The ph.rhdp.io/migrated-repo annotation value from catalog-info.yaml (empty if not set)

## Modules
For each module in nav.adoc order:
- filename (the .adoc filename)
- title
- section_headings (list)
- products_mentioned (list)
- commands_used (list of key CLI commands)
- estimated_duration_min (10-60 based on content length and complexity)
- summary (2-3 sentence description of what the module covers)

## Products and Technologies
Consolidated list of all Red Hat products, operators, and tools found across all modules.

## Infrastructure Signals
What the content implies about infrastructure needs:
- Cluster type (SNO vs multinode signals)
- Operators installed
- VMs referenced
- External services used
- GPU or AI model references
- Storage requirements

## Audience Signals
Difficulty indicators:
- Prerequisites mentioned or assumed
- Complexity of tasks (CLI-heavy, GUI-only, mixed)
- Prior knowledge assumed

## Pre-populated Fields
List all non-empty fields from spec.yaml project section.

Do NOT write any files. Return the report as your final output.
```

## Step 7 — Present analysis to author

Take the content reader's report and present a summary to the author:

> "I've analyzed the imported content. Here's what I found:
>
> **Title:** [title from report]
> **Modules:** [count] modules: [list titles]
> **Products:** [consolidated list]
> **Estimated duration:** [sum of module durations] hours
>
> Does this look right? Anything I should adjust before I generate the spec?"

**Wait for confirmation before proceeding.**

## Step 8 — Spawn Migration Writer agent

Use the Agent tool to spawn a migration writer agent. Pass the content reader's full report and the policy data.

```
Agent prompt:
You are a migration writer for the Publishing House. You take a content analysis report and generate the intake artifacts: design.md, spec.yaml fields, and module outlines.

Project root: <project_root>

## Content Analysis Report
<paste the full content reader report here>

## Policy Data
<paste the policy.json contents — valid products, action verbs, content types, audiences>

## Spec Guidelines
<paste the spec guidelines content>

## Instructions

### How to populate spec.yaml

Read <project_root>/publishing-house/spec.yaml carefully. Every field has an inline comment describing its purpose and valid values (e.g. `# ocp | rhel-vms`, `# beginner | intermediate | advanced`, `# sno | multinode`). The comments ARE the schema — they tell you what each field accepts.

Walk through every field in spec.yaml. For each one:
1. Check if it already has a value (pre-populated by the template) — if so, keep it unless the content analysis clearly contradicts it
2. Check if the content analysis report provides enough information to populate it — if so, set it
3. If the content doesn't provide enough signal — leave the field as-is (empty string, null, or empty list)

Do NOT guess or fabricate values. If the content mentions "OpenShift 4.15" in a prerequisites section, set `ocp_version: "4.15"`. If no OCP version is mentioned anywhere, leave it empty. The same principle applies to every field — populate only what the content supports.

Examples of what the content CAN tell you:
- Operators installed or referenced → products, platform (ocp)
- `oc` commands, console screenshots → platform: ocp
- VM provisioning, `ssh` into hosts → platform: rhel-vms
- Number of workers mentioned in setup → worker_count, cluster_type
- AI/ML model references → ai_requirement, ai_model_name
- External URLs called → external_services list
- AAP playbooks or controller UI → aap_version

Examples of what the content usually CANNOT tell you:
- Exact CPU/RAM sizing per node (unless explicitly documented)
- Cost estimates
- Topology decisions (shared vs per-student)
- Non-GA access plans

### Phase 1 — Design Generation
Follow the procedure in <project_root_skills>/skills/migrate/procedures/01-design-from-content.md

Read the design.md template at <project_root>/publishing-house/spec/design.md and fill in every section using the content analysis report. For the Infrastructure Requirements section, fill in what the content supports and mark unknown fields as "TBD — confirmed in infrastructure phase".

After writing design.md, update spec.yaml — walk through every field and populate what the content analysis supports. Commit:
```bash
git add publishing-house/spec/design.md publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "feat: design doc and spec populated from imported content" 2>/dev/null || true
```

Validate design.md against spec guidelines:
- All 11 required sections present
- Learning objectives use valid action verbs from policy
- No unfilled template placeholders
- Module durations in 10-60 minute range
- Module Map table has at least one row

### Phase 2 — Module Outlines
Read the module outline template at <project_root>/publishing-house/spec/module-outline-template.md.

For each module in the Module Map table:

1. Generate one outline file:
   - Output directory: <project_root>/publishing-house/spec/modules/
   - Naming: module-01-<short-title>.md, module-02-<short-title>.md
   - Follow the template structure exactly
   - Derive content from the corresponding module in the content analysis report

2. Rename the content file to match the outline name:
   - Old: <project_root>/content/modules/ROOT/pages/old-name.adoc
   - New: <project_root>/content/modules/ROOT/pages/module-01-<short-title>.adoc
   - The base name MUST match exactly (e.g., module-01-pipeline-setup)

3. Rename the runtime-automation folder to match:
   - Old: <project_root>/runtime-automation/old-name/
   - New: <project_root>/runtime-automation/module-01-<short-title>/
   - Skip if the folder doesn't exist

4. Update <project_root>/content/modules/ROOT/nav.adoc:
   - Replace old filename references with new names
   - Example: `* xref:old-name.adoc[Old Title]` → `* xref:module-01-pipeline-setup.adoc[Pipeline Setup]`

Note: this does NOT touch `ui-config.yml`, even though a migrated repo's copy of it (imported
from the source repo) already exists at this point and still has the old names. It's aligned
later, during the `development` stage — `config-helper` Route A step A.1b matches its
`antora.modules` names to whatever renames happened here.

After writing outlines and renaming files:

1. Add `status: not_started` to each module in `spec.modules` (if not already present). Example:
   ```yaml
   modules:
     - id: module-01
       title: "Pipeline Setup"
       duration_min: 30
       status: not_started
   ```

2. Generate summaries for spec.yaml:
   - approval_checklist.content.design_overview (2-3 sentences)
   - approval_checklist.content.module_summaries (1-2 sentences per module)

Commit:
```bash
git add publishing-house/spec/modules/ publishing-house/spec.yaml
git add content/modules/ROOT/pages/ content/modules/ROOT/nav.adoc
git add runtime-automation/
git diff --cached --quiet || git commit -m "feat: module outlines generated and content files renamed to match" 2>/dev/null || true
```

Do NOT proceed past Phase 2. Return a summary of what was written:
- List of files created/modified
- Design doc section count
- Module outline count
- spec.yaml fields populated vs left empty
- Any validation warnings
```

## Step 9 — RCARS Vetting

After the migration writer completes, run RCARS vetting yourself (not in an agent — this requires API calls via tools scripts).

Follow `@rhdp-publishing-house/skills/intake/procedures/03b-rcars-vetting.md`.
If offline → skip with warning.

**Migrated repo filtering:** Before presenting RCARS candidates, read `catalog-info.yaml` and extract the `ph.rhdp.io/migrated-repo` annotation. If it has a value, derive the repo name (last path segment of the URL) and filter out any RCARS candidate whose `ci_name` contains that repo name. The migrated lab is the one being imported — flagging it as overlap is meaningless.

After completion: "RCARS vetting complete. Next: infrastructure confirmation. **(2 phases remaining)**"

## Step 10 — Infrastructure Confirmation

Follow `@rhdp-publishing-house/skills/intake/procedures/05-infrastructure.md`.

## Step 11 — Finalize + Submit

Follow `@rhdp-publishing-house/skills/intake/procedures/06-finalize-and-submit.md`.

After Step 11 completes, **return to the orchestrator** (if dispatched) or **STOP** (if invoked directly).

## Pre-populated Fields

The migration template pre-populates some `project.*` fields in spec.yaml (slug, owner_email, content_type, deployment_mode, etc.). The writer agent discovers these by reading spec.yaml — it does not use a hardcoded list. Any field that already has a value is kept unless the content analysis clearly contradicts it and the author confirms the change.
