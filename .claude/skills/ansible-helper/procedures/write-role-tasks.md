# Write Role Tasks — A5 Sub-procedure

Called from `procedures/new-role.md` when the author wants tasks written automatically.

## Step 1 — Get detailed description

Ask:
> "Describe what the role needs to do in detail: target system, what it configures,
> any variables or conditions, and the expected end state."

Wait for the description.

## Step 2 — Detect role type and load reference

Based on the role name and description, determine which modules to use:

- Targets **AAP Controller** (organizations, teams, users, credentials, inventories,
  projects, job templates, workflow job templates, notifications) →
  read `.claude/skills/role/references/ansible-controller-modules.md`

- Targets **EDA Controller** (projects, credentials, decision environments,
  event streams, rulebook activations) →
  read `.claude/skills/role/references/ansible-eda-modules.md`

- Targets **both** → read both reference files above.

- Targets **other** (RHEL config, packages, services, files, etc.) →
  use standard `ansible.builtin` modules; no reference file needed.

If the reference files above do not exist, generate tasks from general Ansible
knowledge using the correct module FQCNs for the target system.

## Step 3 — Write tasks/main.yml

Use the dispatcher pattern where applicable: include sub-task files per component type
with `when: <list_var> | length > 0`. For simpler roles write tasks directly.

Rules:
- Prefix every file with `#SPDX-License-Identifier: MIT-0`
- Include `controller_host`, `controller_username`, `controller_password`,
  `validate_certs` on every AAP/EDA module call
- Use `loop` with `loop_control.label` for all list-based configuration
- Use `no_log: true` on tasks that handle credentials or passwords
- Use `| default(omit)` for optional module parameters

## Step 4 — Write defaults/main.yml

Populate with every variable the tasks reference:
- Connection variables (`controller_host`, `controller_username`,
  `controller_password`, `validate_certs`)
- One empty list per component type, defaulting to `[]`
- Inline YAML comments showing an example entry for each list

## Step 5 — Commit

```bash
git add automation/ansible/roles/<role_name>/
git diff --cached --quiet || git commit --author="Mitesh Sharma <mitsharm@redhat.com>" \
  -m "feat: write <role_name> role tasks"
```

Return to `procedures/new-role.md` at step A6.
