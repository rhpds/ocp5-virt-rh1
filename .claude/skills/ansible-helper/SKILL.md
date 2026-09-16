---
name: rhdp-publishing-house:ansible-helper
description: This skill should be used when the user asks to "create an ansible role", "add a role to the ansible collection", "new ansible role", "scaffold a role in automation/ansible", "import roles from git", or "migrate ansible roles into the collection". Creates new roles from scratch or imports existing roles from a Git repository into the automation/ansible/ collection.
context: main
---

# Ansible Helper

You create or import Ansible roles inside the project's `automation/ansible/` Ansible Collection.

See @rhdp-publishing-house/skills/common/user-interaction.md for how to present multi-option choices to the author.

## Mode Detection

Detect the mode before doing anything else:

```bash
test -f catalog-info.yaml && test -f publishing-house/spec.yaml && echo "ph" || echo "standalone"
```

- `ph` → **Publishing House mode**. Start at Step 1.
- `standalone` → **Standalone mode**. Skip to Step 3.

---

## Steps 1–2 — Publishing House mode only

### Step 1 — Pre-flight

**Detect Python:**
```bash
command -v python3 >/dev/null 2>&1 && echo "python3" || command -v python >/dev/null 2>&1 && echo "python" || echo "none"
```
If `none` → show: "Python 3 is required. Install it and retry." **STOP.**

**Verify Publishing House project:**
```bash
test -f catalog-info.yaml && test -f publishing-house/spec.yaml && echo "ok" || echo "missing"
```
If `missing` → show: "Not a Publishing House project root. Navigate to the project root and retry." **STOP.**

**Read project identity:**
```bash
python3 -c "
import yaml
from pathlib import Path
spec = yaml.safe_load(Path('publishing-house/spec.yaml').read_text()) or {}
p = spec.get('project', {})
slug = p.get('slug', '')
email = p.get('owner_email', '')
print(f'namespace:{slug.replace(\"-\",\"_\")}')
print(f'slug:{slug}')
print(f'owner_email:{email}')
"
```
Store: `namespace`, `slug`, `owner_email`.

### Step 2 — Workflow check

**RULE: This sequence runs every invocation. No exceptions. No skipping.**

**2a.** Get workflow data:
```bash
python publishing-house/tools/ph-workflow-data.py
```
If this fails → set `offline_mode = true`, skip to Step 3.
If this succeeds → extract `workflow_id`. Set `offline_mode = false`.

**2b.** Get workflow state (skip if offline):
```bash
python publishing-house/tools/ph-workflow-state.py WORKFLOW_ID
```
If stage is not `development` → STOP. Tell the author this skill runs during the development stage.
If offline → assume `development`.

**2c.** Sync (skip if offline):
```bash
python publishing-house/tools/ph-sync.py
```

---

## Steps 3+ — Both modes

### Step 3 — Verify collection and list roles

**Verify collection directory:**
```bash
test -d automation/ansible && echo "EXISTS" || echo "MISSING"
test -d automation/ansible/roles && echo "ROLES_OK" || echo "ROLES_MISSING"
```
If `automation/ansible` is MISSING → show:
> "The `automation/ansible/` collection directory does not exist. Create it first using
> `ansible-galaxy collection init <namespace>.ansible --init-path automation/`
> then re-run this skill."

**STOP.**

If `automation/ansible/roles` is ROLES_MISSING → silently create it:
```bash
mkdir -p automation/ansible/roles
```

**List existing roles:**
```bash
ls -1 automation/ansible/roles/ 2>/dev/null
```
Store the result as `existing_roles`. If the directory is empty, `existing_roles` is an empty list.

---

## Step 4 — Show inventory and choose path

**Immediately after pre-flight. Do NOT wait.**

If `existing_roles` is non-empty, show the current role inventory before the menu:

> "**Existing roles in `automation/ansible/roles/`:**
>
>  1. `role_name_a`
>  2. `role_name_b`
>  ...
>
> Total: N role(s)"

Then ask:

> "What would you like to do?
>
> 1. **New role** — write a fresh Ansible role from scratch inside the collection
> 2. **Import from Git** — pull one or more existing roles from a Git repository"

If `existing_roles` is empty, skip the inventory block and show only the menu.

---

## Dispatch

- **Option 1** → follow `procedures/new-role.md`
- **Option 2** → follow `procedures/migrate-roles.md`

## Completion confirmation (Publishing House mode only)

Skip this step in standalone mode.

After the procedure completes and the user declines to add/import more roles, ask:

> "Ansible automation work is done. Is automation complete, or do you need to do more work?"
> 1. Mark automation complete
> 2. Back to dashboard (I'll finish later)

- **1** →
  1. Set `development.automation.ansible.status: complete` in `publishing-house/spec.yaml`
  2. Commit and push:
     ```bash
     git add publishing-house/spec.yaml
     git commit -m "feat: mark ansible automation complete"
     git push
     ```
  3. Check if **all** applicable automation children are now `complete` (read `project.automation_type`
     from spec.yaml — if `both`, check that the other child is also complete). If all complete, close the
     Jira ticket:
     ```bash
     python publishing-house/tools/ph-task-complete.py write-automation
     ```
     If other children are still incomplete, do not close the ticket.
  4. Confirm: "Ansible automation marked complete. Returning to development dashboard."
- **2** → Confirm: "Returning to development dashboard. You can come back to Ansible automation anytime."
