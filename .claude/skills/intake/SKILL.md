---
name: rhdp-publishing-house:intake
description: This skill should be used when the user asks to "create a spec", "write a design doc", "start a new lab project", "I have an idea for a lab", "I have a Jira issue with requirements", or "pull requirements from Jira". It handles intake for RHDP Publishing House projects.
context: main
---

# Intake Agent

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You handle the intake phase of the Publishing House lifecycle. This skill is
self-sufficient — it works whether dispatched by the orchestrator or invoked directly.

## Core Principles

1. **Template-driven.** You learn what to fill in by reading the project's `design.md` (placeholder sections) and `spec.yaml` (inline comments with valid values). You do NOT have your own template copies.

2. **Conversational.** You suggest and confirm. You never read from a script or ask rigid questions in a fixed order. You have a natural conversation about the author's idea and capture structured data from it.

3. **Write per phase.** Within a phase, focus on the conversation. At the end of each phase, write all captured fields to spec.yaml and design.md in one commit. Each phase is a checkpoint — if the session ends, work up to the last completed phase is saved.

4. **Author reviews design.md only.** spec.yaml is written silently at each checkpoint. The author reviews and approves the human-readable design doc, not the YAML.

## Tool Boundaries

**Do NOT use** Central API tools directly. You work locally: read files, write specs, update spec.yaml.

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

**4d.** If `unresolved_rejections` > 0 → follow `procedures/00-rejection-handler.md`. After the rejection handler completes, continue with normal intake (Step 5 onward).

**4e.** Fetch pre-intake data from Jira (skip if offline or if `project.jira_ticket` is empty):
```bash
python publishing-house/tools/ph-preintake-data.py > ~/.config/publishing-house/preintake-data.json 2>/dev/null || true
```

If successful, `~/.config/publishing-house/preintake-data.json` contains:
- `epic_key` — Jira epic key
- `description` — Raw text from the epic description (converted from ADF)

Read this file before starting phases. The description contains pre-intake information that should inform your questions and populate spec.yaml fields during discovery and design phases.

### Step 5 — Load policy and project files

1. Fetch validation policy:
   ```bash
   python publishing-house/tools/ph-policy.py
   ```
   If it fails and offline → check for `publishing-house/policy.json` as a static fallback.
   If no policy available at all → warn but continue (validation happens at submission).

2. Read `~/.config/publishing-house/policy.json` if it exists. Use these lists throughout:
   - `valid_content_types` — constrain content type choices
   - `valid_audiences` — constrain audience choices
   - `products` (with `aliases`) — validate product names
   - `action_verbs_valid` — learning objectives must start with one of these
   - `action_verbs_rejected` — reject objectives starting with these

3. Read `publishing-house/spec.yaml` — understand the structure, valid values from inline comments, and which fields are already populated

4. Read `publishing-house/spec/design.md` — understand the section structure from the placeholders. This is the template you will fill in.

5. Read spec guidelines at `@rhdp-publishing-house/skills/intake/references/spec-guidelines.md`

6. Read the module outline template from the project repo at `publishing-house/spec/module-outline-template.md`

## Dispatch

Rejections are already handled in Step 4d. This section determines the entry path when there are no rejections (or after the rejection handler completes).

Read `publishing-house/spec/design.md`. Check whether it still has `[placeholder]` markers
or contains real content.

Read `publishing-house/spec.yaml`. Check whether `spec.title`, `spec.learning_objectives`,
and `spec.modules` are populated.

**If design.md is mostly placeholders AND spec fields are empty:**

Read `~/.config/publishing-house/preintake-data.json` if it exists. The description field contains pre-intake information from the Jira epic. Use this data as input when going through the phases.

Proceed with Phase 1 (Discovery) through Phase 6. The pre-intake data will inform which fields are already populated and which questions to skip.

**If design.md has real content (not placeholders) → resume mode:**

Determine which phases are complete:
- design.md filled in → Phase 2 done
- RCARS results in spec.yaml (`approval_checklist.content.rcars_top_matches` non-empty) → Phase 3 done
- Module outline files exist in `publishing-house/spec/modules/` → Phase 4 done
- Infrastructure fields populated in spec.yaml → Phase 5 done

Show a summary of what's already captured and pick up at the next incomplete phase.

## Phase Flow

Show at the start:
> "Six phases: discovery, design, RCARS vetting, module outlines, infrastructure, and submission."

### Phase 1 — Discovery
Follow `procedures/02-discovery.md`.
After completion: "Discovery complete. Next: design doc. **(4 phases remaining)**"

### Phase 2 — Design Generation
Follow `procedures/03-design-doc.md`.
After completion: "Design doc complete and validated. Next: RCARS vetting. **(3 phases remaining)**"

### Phase 3 — RCARS Vetting
Follow `procedures/03b-rcars-vetting.md`.
If offline → skip with warning: "RCARS vetting skipped (offline). This will run at submission."
After completion: "RCARS vetting complete. Next: module outlines. **(2 phases remaining)**"

### Phase 4 — Module Outlines
Follow `procedures/04-module-outlines.md`.
After completion: "Module outlines complete. Next: infrastructure confirmation. **(1 phase remaining)**"

### Phase 5 — Infrastructure Confirmation
Follow `procedures/05-infrastructure.md`.

### Phase 6 — Finalize + Submit
Follow `procedures/06-finalize-and-submit.md`.

After Phase 6 completes, **return to the orchestrator** (if dispatched) or **STOP** (if invoked directly).

## Pre-populated Fields

Before asking questions, check spec.yaml for fields already set by the RHDH template:
- `project.slug` — project identifier
- `project.owner_email` — author email
- `project.content_type` — lab or demo
- `project.deployment_mode` — rhdp_published or self_published
- `project.initiative_key` — e.g., rh1_2027
- `project.showroom_type` — classic or zero_touch
- `project.description` — project description from RHDH form

Additionally, if `~/.config/publishing-house/preintake-data.json` exists (from Step 4e), the `description` field contains pre-intake information that can populate:
- `spec.title` — Asset Title
- `spec.modules[]` — Content Outline
- `spec.learning_objectives[]` — Learning Objectives
- `project.content_type` — Lab or Demo
- `project.automation_type` — Automation Type

**Skip asking about any field that already has a value.**
