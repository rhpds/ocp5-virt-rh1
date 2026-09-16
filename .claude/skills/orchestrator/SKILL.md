---
name: rhdp-publishing-house
description: This skill should be used when the user invokes "/rhdp-publishing-house", asks to "start a publishing house project", "check project status", or "what's next on my lab". Checks workflow state and dispatches to the appropriate stage skill.
context: main
---

# RHDP Publishing House — Orchestrator

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You are a thin dispatcher. You check the workflow state and hand off to the right skill. You do NOT own intake logic, spec writing, or development work.

See @rhdp-publishing-house/skills/orchestrator/references/gate-language.md for how to present stage status.
See @rhdp-publishing-house/skills/orchestrator/references/session-protocol.md for session start/end protocol.
See @rhdp-publishing-house/skills/orchestrator/references/spec-rules.md for spec.yaml rules.
See @rhdp-publishing-house/skills/common/user-interaction.md for how to present multi-option choices to the author.

## Tool Boundaries

**Do NOT use** MCP tools or Central API tools directly. All external interactions go through `publishing-house/tools/` scripts.
**Do NOT use** the browser or external network calls — tools scripts handle all API communication.

## Steps 1–3 — Pre-flight

Follow @rhdp-publishing-house/skills/common/pre-flight.md (Steps 1–3: verify project, read identity, check auth).

## Step 4 — Get workflow state

Run silently:
```bash
python publishing-house/tools/ph-workflow-data.py
```

Extract `workflow_id` from the output.

Then run:
```bash
python publishing-house/tools/ph-workflow-state.py WORKFLOW_ID
```
Replace WORKFLOW_ID with the extracted `workflow_id`. Extract `stage` from the output.

Both scripts are read-only — they never write files.

## Step 5 — Dispatch

**RULE: Dispatch based on `stage` only. No interpretation, no session context, no reasoning about what happened previously.** The stage returned by the API is the truth. Map it to a skill and dispatch. That is your only job.

This is a loop. After a skill returns, re-run Step 4 (both scripts), extract the new stage, and continue.

```
Loop:
  intake       → read project.intake_type from publishing-house/spec.yaml
                  if intake_type == "migration" → dispatch rhdp-publishing-house:migrate
                  else → dispatch rhdp-publishing-house:intake
  development  → dispatch rhdp-publishing-house:development
  content_review / infra_review / staging → show review status, STOP
  testing      → show testing status, STOP
  published    → show published status, STOP
```

### Stage status messages

**content_review / infra_review / staging:**
> Spec submitted. Three pending stages must complete before advancing to Development:
> - **Content Review** — design spec and module outlines
> - **Infra Review** — environment and automation requirements
> - **Staging** — an RHDP content developer builds the base CI
>
> Reviewers approve from the RHDH Publishing House portal. Content developers complete staging from the portal.

**development:**
> Development is now active. What do you need help with?

**testing:**
> Development is complete. The project is undergoing testing before release.

**published:**
> This project is published.

## Rules

- Never tell the author to run any script except opening the portal URL during first-time key setup
- ALWAYS show the portal URL in the conversation — never rely solely on `open` working (DevSpaces has no browser)
- **`project_id`** comes from `spec.yaml` `project.slug`
- **`central_url`** comes from the **Central** link in `catalog-info.yaml` (cached in `~/.config/publishing-house/auth.json`)
- Stage is always read from the Central API via `ph-workflow-data.py` and `ph-workflow-state.py`
- The orchestrator dispatches skills but does not own submission or advancement — each skill handles its own API calls
