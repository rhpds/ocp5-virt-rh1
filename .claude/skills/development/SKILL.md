---
name: rhdp-publishing-house:development
description: This skill should be used when the user asks to "set up showroom", "configure showroom tabs", "create site.yml", "scaffold the showroom structure", "add a tab", "review my showroom config", "check site.yml", "validate ui-config.yml", "development dashboard", "what's next to develop", or "submit to central". Handles showroom scaffolding, config review, workstream status tracking, and submission to Central during the development stage.
context: main
---

# Development Agent

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You handle the development phase of the Publishing House lifecycle. After scaffolding, this skill
does exactly three things: scaffold the showroom structure, track module status in `spec.yaml`, and
submit to Central when the author says all content is complete. Writing, reviewing, and automation
are optional helper skills the author may use — or not — entirely at their own discretion.

## Tool Boundaries

**Do NOT use** Central API tools directly. You work locally: read files, write content, update spec.yaml.

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
If stage is not `development` → STOP. This is the only condition that stops the skill.
If offline → assume `development`.

**4c.** Sync (skip if offline):
```bash
python publishing-house/tools/ph-sync.py
```
Extract `unresolved_rejections` from the output. Commit any changes:
```bash
git add publishing-house/spec.yaml catalog-info.yaml
git diff --cached --quiet || git commit -m "feat: sync workflow data from Central API" 2>/dev/null || true
```

**4d.** If `unresolved_rejections` > 0 → show the unresolved rejection reasons to the author (read from `approval_checklist.content.rejections` and `approval_checklist.infra.rejections` in spec.yaml). Help the author address each one, then continue with normal development.

### Step 5 — Read project context

1. Read `publishing-house/spec.yaml` for project metadata and spec data
2. Read `publishing-house/spec/design.md` for the design spec
3. Read module outlines in `publishing-house/spec/modules/`

## Dispatch

### Step 1 — Scaffold check gate

Follow `procedures/config-reviewer.md` automatically against the project's content directory.

- **PASS** → proceed to Step 2
- **FAIL, project not yet scaffolded** (`.scaffolds/` directory still present) → the author has no way
  to know the required directory/file structure yet — that's exactly what scaffolding is for. Do NOT
  ask permission or offer to let them handle it themselves. Briefly report what's missing, then go
  straight into `procedures/config-helper.md` (which detects `.scaffolds/` and runs Route A
  automatically). Route A's own scaffold-plan confirmation is the only checkpoint the author needs to
  see. Proceed to Step 2 once it returns.
- **FAIL, project already scaffolded** (`.scaffolds/` is gone, but config is genuinely missing or
  invalid) → the author already has the real structure and may be mid-edit intentionally, so report
  the specific issues and ask:
  > "The showroom scaffold has issues that need to be resolved first.
  > 1. Help me fix them
  > 2. I'll handle it myself"
  - **1** → follow `procedures/config-helper.md` to fix the scaffold, then proceed to Step 2
  - **2** → STOP. Do not proceed until the author says the scaffold is ready.

### Step 2 — In-progress check

Before showing the dashboard, check for any `in_progress` work and ask the author about each one.

**2a. Modules** — For each module with `status: in_progress`, ask:

> "Module N — *[title]* is in progress. Are you done with it?"
> 1. Yes, mark it complete
> 2. No, still working on it

- **1** →
  1. Verify the module's `.adoc` file exists in `content/modules/ROOT/pages/`. If not, warn the author
     but still allow marking complete if they confirm.
  2. Set `status: complete` in spec.yaml
  3. Commit and push:
     ```bash
     git add publishing-house/spec.yaml
     git commit -m "feat: mark module N complete — [title]"
     git push
     ```
  4. Close the Jira ticket (best-effort):
     ```bash
     python publishing-house/tools/ph-task-complete.py module-NN
     ```
- **2** → leave as `in_progress`, move to next.

Ask about all `in_progress` modules sequentially before proceeding.

**2b. Automation** — Read `project.automation_type` from `spec.yaml`. For each applicable child
(`gitops` if type is `gitops` or `both`; `ansible` if type is `ansible` or `both`), check
`development.automation.<child>.status`.

If a child's status is `in_progress`, ask:

> "[Child] automation is in progress. Are you done with it?"
> 1. Yes, mark it complete
> 2. No, still working on it

- **1** →
  1. Set `development.automation.<child>.status: complete` in spec.yaml
  2. Commit and push
  3. Check if **all** applicable automation children are now `complete`. If yes, close the Jira ticket:
     `python publishing-house/tools/ph-task-complete.py write-automation`
     If other children are still incomplete, do not close the ticket.
- **2** → leave as `in_progress`, move on.

Ask about each applicable child sequentially before proceeding.

**2c. E2E Tests** — If `development.e2e.status` is `in_progress`, ask:

> "E2E Tests are in progress. Are you done with them?"
> 1. Yes, mark complete
> 2. No, still working on it

- **1** →
  1. Set `development.e2e.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-e2e-tests`
- **2** → leave as `in_progress`, move on.

**2d. Health Check** — If `development.healthCheck.status` is `in_progress`, ask:

> "Health Check is in progress. Are you done with it?"
> 1. Yes, mark complete
> 2. No, still working on it

- **1** →
  1. Set `development.healthCheck.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-health-check`
- **2** → leave as `in_progress`, move on.

After all checks, proceed to Step 3.

### Step 3 — Development Dashboard

Read all workstream statuses from `spec.yaml` (updated after Step 2) and present the dashboard.
This runs every time — it is the central hub of the development phase.

Only show **incomplete** workstreams (not_started or in_progress). Completed workstreams are hidden.

Read from `spec.yaml`:
- Module statuses from `spec.modules[*].status`
- `project.automation_type` — determines which automation children exist
- `development.automation.gitops.status` (if `automation_type` is `gitops` or `both`)
- `development.automation.ansible.status` (if `automation_type` is `ansible` or `both`)
- `development.e2e.status`
- `development.healthCheck.status`

Build the dashboard dynamically based on what is still incomplete:

> **Development Dashboard**
>
> | # | Workstream | Status |
> |---|------------|--------|

Include these rows **only if incomplete**:
- **Modules** — show if any module is not `complete` (display "N of M complete")
- **GitOps Automation** — show if `automation_type` is `gitops` or `both` AND `development.automation.gitops.status` is not `complete`
- **Ansible Automation** — show if `automation_type` is `ansible` or `both` AND `development.automation.ansible.status` is not `complete`
- **E2E Tests** — show if `development.e2e.status` is not `complete`
- **Health Check** — show if `development.healthCheck.status` is not `complete`

Always include:
- **Showroom Config** — always shown (set up / review)

Number rows sequentially starting at 1 based on what is shown.

> Type a number to work on that item.

**If this is the first visit** (all modules `not_started` and all development fields `not_started`),
append:

> Your showroom is scaffolded and ready for content. Each workstream has optional AI helpers
> available when you select it, or you can write everything yourself — the helpers are a starting
> point, not a substitute for your own review. Don't rely on AI output alone; read through and
> verify whatever a helper produces before marking a workstream complete. When required workstreams
> are done, I'll submit to Central.

### Step 4 — Submission gate

**Trigger:** All required workstreams are complete:
- All modules have `status: complete`
- All applicable automation children have `status: complete` (gitops and/or ansible based on `project.automation_type`)
- `development.e2e.status` is `complete`
- `development.healthCheck.status` is `complete`

**Required complete →**
> "All workstreams are complete.
> 1. Yes, submit development
> 2. No, not yet"

- **1** →
  1. Run `python publishing-house/tools/ph-development.py`. If it fails, STOP and show the error.
  2. Confirm: "Development submitted to Central — workflow advanced to review stage."
- **2** → return to dashboard.

**Required not complete →** do not offer submission. Show the dashboard with outstanding items.

### Step 5 — Workstream selection and dispatch

Based on the user's number selection from the dashboard.
The dashboard rows are dynamic — match the user's selection to the workstream shown at that number.

#### Option 1 — Modules

Always list **every** module from `spec.modules`, in order, regardless of status. Assign
sequential selectable numbers (1, 2, 3, ...) only to modules that are `not_started` or
`in_progress`. For modules with `status: complete`, show `—` in the `#` column instead of a
number — there is no digit for the author to type to select it. Number "Back to dashboard" as
`(count of selectable modules) + 1`.

> **Modules**
>
> | # | Module | Status |
> |---|--------|--------|
> | 1 | [title] | not_started / in_progress |
> | — | [title] | complete |
> | 2 | [title] | not_started / in_progress |
> | ... | ... | ... |
> | N+1 | Back to dashboard | |
>
> Type a number to select a module. Completed modules are shown for reference and can't be selected.

When the author selects a module:

1. If `not_started`:
   - Set `status: in_progress` in spec.yaml
   - Create an empty `.adoc` stub at `content/modules/ROOT/pages/[filename].adoc` if it doesn't exist:
     ```adoc
     = [Module Title]
     ```
   - Commit:
     ```bash
     git add publishing-house/spec.yaml content/modules/ROOT/pages/[filename].adoc
     git commit -m "feat: start module N — [title]"
     ```

2. Show the action menu:

> **Module N — [title]**
>
> | # | Option |
> |---|--------|
> | 1 | Write it myself |
> | 2 | Use AI writer helper |
> | 3 | Back to dashboard |

**Option 1 — Write it myself:**
  Tell the author:
  > "Write your content in `content/modules/ROOT/pages/[filename].adoc`."
  > 1. Done — mark module complete
  > 2. Back to dashboard (I'll finish later)

  - **1** → mark module complete:
    1. Set `status: complete` in spec.yaml
    2. Commit and push:
       ```bash
       git add publishing-house/spec.yaml
       git commit -m "feat: mark module N complete — [title]"
       git push
       ```
    3. Close the Jira ticket (best-effort):
       ```bash
       python publishing-house/tools/ph-task-complete.py module-NN
       ```
    4. Return to dashboard.
  - **2** → return to dashboard (module stays `in_progress`, will be caught by Step 2 next time).

**Option 2 — AI writer helper:**
  Dispatch to `rhdp-publishing-house:writer-helper` skill. When it returns, mark module complete:
  1. Verify the module's `.adoc` file exists in `content/modules/ROOT/pages/`. If not, warn the author
     but still allow marking complete if they confirm.
  2. Set `status: complete` in spec.yaml
  3. Commit and push:
     ```bash
     git add publishing-house/spec.yaml
     git commit -m "feat: mark module N complete — [title]"
     git push
     ```
  4. Close the Jira ticket (best-effort):
     ```bash
     python publishing-house/tools/ph-task-complete.py module-NN
     ```
  5. Return to dashboard.

**Option 3 — Back:** Return to dashboard.

#### Automation workstream — GitOps

Shown when the user selects the **GitOps Automation** row from the dashboard.

If `development.automation.gitops.status` is already `complete`:
> "GitOps automation is already complete. Would you like to reopen it?"
> 1. Reopen (set back to `in_progress`)
> 2. Back to dashboard

Otherwise, check if `automation/gitops/` directory exists.

**If `automation/gitops/` directory does not exist:**
> "The GitOps automation skeleton hasn't been created yet. Would you like me to scaffold it?"
> 1. Yes, scaffold automation directories
> 2. Back to dashboard

- **1** → follow `procedures/config-helper.md` (Automation Scaffolding section). When it completes, continue below.
- **2** → return to dashboard.

**If `automation/gitops/` directory exists**, show:

> **GitOps Automation**
>
> | # | Option |
> |---|--------|
> | 1 | Use GitOps helper (populates Helm charts with workloads) |
> | 2 | Do it myself |
> | 3 | Back to dashboard |

- **1** → Set `development.automation.gitops.status: in_progress` if currently `not_started`, commit, then dispatch to `rhdp-publishing-house:gitops-helper` skill. When it returns, return to dashboard.
- **2** → Set `development.automation.gitops.status: in_progress` if currently `not_started`, commit. Tell the author:
  > "Work on your GitOps automation in `automation/gitops/bootstrap-infra/` (and `bootstrap-tenant/` if it exists). When you're done, come back and I'll mark it complete."
  Return to dashboard.
- **3** → return to dashboard.

#### Automation workstream — Ansible

Shown when the user selects the **Ansible Automation** row from the dashboard.

If `development.automation.ansible.status` is already `complete`:
> "Ansible automation is already complete. Would you like to reopen it?"
> 1. Reopen (set back to `in_progress`)
> 2. Back to dashboard

Otherwise, check if `automation/ansible/` directory exists.

**If `automation/ansible/` directory does not exist:**
> "The Ansible automation skeleton hasn't been created yet. Would you like me to scaffold it?"
> 1. Yes, scaffold automation directories
> 2. Back to dashboard

- **1** → follow `procedures/config-helper.md` (Automation Scaffolding section). When it completes, continue below.
- **2** → return to dashboard.

**If `automation/ansible/` directory exists**, show:

> **Ansible Automation**
>
> | # | Option |
> |---|--------|
> | 1 | Use Ansible helper |
> | 2 | Do it myself |
> | 3 | Back to dashboard |

- **1** → Set `development.automation.ansible.status: in_progress` if currently `not_started`, commit, then dispatch to `rhdp-publishing-house:ansible-helper` skill. When it returns, return to dashboard.
- **2** → Set `development.automation.ansible.status: in_progress` if currently `not_started`, commit. Tell the author:
  > "Build your Ansible automation manually in `automation/ansible/`. When you're done, come back and I'll mark it complete."
  Return to dashboard.
- **3** → return to dashboard.

#### Option — E2E Tests

If `development.e2e.status` is already `complete`:
> "E2E tests are already complete. Would you like to reopen?"
> 1. Reopen (set back to `in_progress`)
> 2. Back to dashboard

Otherwise, set `development.e2e.status: in_progress` if currently `not_started`, commit, then show:

> **E2E Tests**
>
> Write your E2E tests manually in `qa-automation/e2e.yml`. See the
> [E2E Tests docs](https://rhpds.github.io/rhdp-publishing-house/user/qa-automation/#e2eyml) for
> requirements and guidance.
>
> | # | Option |
> |---|--------|
> | 1 | Mark E2E tests complete |
> | 2 | Back to dashboard |

- **1** →
  1. Set `development.e2e.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-e2e-tests`
  4. Return to dashboard.
- **2** → return to dashboard.

#### Option — Health Check

If `development.healthCheck.status` is already `complete`:
> "Health check is already complete. Would you like to reopen?"
> 1. Reopen (set back to `in_progress`)
> 2. Back to dashboard

Otherwise, set `development.healthCheck.status: in_progress` if currently `not_started`, commit, then show:

> **Health Check**
>
> Write your health check playbook manually in `qa-automation/healthcheck.yml`. See the
> [Health Check docs](https://rhpds.github.io/rhdp-publishing-house/user/qa-automation/#healthcheckyml)
> for requirements and guidance.
>
> | # | Option |
> |---|--------|
> | 1 | Mark health check complete |
> | 2 | Back to dashboard |

- **1** →
  1. Set `development.healthCheck.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-health-check`
  4. Return to dashboard.
- **2** → return to dashboard.

#### Option — Showroom Config

> | # | Option |
> |---|--------|
> | 1 | Set up showroom (config-helper) |
> | 2 | Review showroom config (config-reviewer) |
> | 3 | Back to dashboard |

- **1** → follow `procedures/config-helper.md`
- **2** → follow `procedures/config-reviewer.md`
- **3** → return to dashboard.

## Rules

- Never tell the author to run any script
- The development skill owns scaffolding, workstream status tracking, and Central submission
- Writing, reviewing, and building automation are optional helper skills
  (`rhdp-publishing-house:writer-helper`, `rhdp-publishing-house:reviewer-helper`,
  `rhdp-publishing-house:gitops-helper`) that the author invokes independently — they're a
  starting point, not a substitute for the author's own manual review
- `config-helper.md` and `config-reviewer.md` are the only procedures this skill dispatches to
- Status transitions: `not_started` → `in_progress` (on selection) → `complete` (on explicit human confirmation only)
- **NEVER mark a workstream complete without the author explicitly saying it is done**
- When adding a new module to `spec.modules`, always set `status: not_started`:
  ```yaml
  - id: module-NN
    title: "Module Title"
    duration_min: 30
    status: not_started
  ```
