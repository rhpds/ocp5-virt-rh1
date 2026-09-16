# New Role — Path A

Creates a new Ansible role skeleton inside `automation/ansible/roles/`.

Read `references/collection-structure.md` for canonical role file formats and stub content.

## A1 — Role name

Ask:
> "What should the role be called? (snake_case, e.g. `configure_aap`, `deploy_nginx`)"

Wait. Silently convert any hyphens or spaces to underscores. If empty or unclear → ask again.

## A2 — Role purpose

Ask:
> "What is this role responsible for? Tell me what it should configure, deploy, or manage —
> the more detail you give, the better I can write the tasks later. (A sentence or two is fine.)"

Wait. Extract `role_description`.

## A3 — Duplicate check

```bash
test -d automation/ansible/roles/<role_name> && echo "EXISTS" || echo "NEW"
```

If EXISTS → ask:
> "`automation/ansible/roles/<role_name>/` already exists. Continue anyway (won't overwrite
> existing files) or pick a different name?"

If different name → back to A1. If continue → proceed.

## A4 — Scaffold role skeleton

Create directories:
```bash
mkdir -p automation/ansible/roles/<role_name>/{tasks,defaults,meta}
```

Write each stub file using the templates in `references/collection-structure.md`.
Substitute `<role_name>`, `<role_description>`, `<owner_email>`, `<namespace>` everywhere.

Commit:
```bash
git add automation/ansible/roles/<role_name>/
git diff --cached --quiet || git commit --author="Mitesh Sharma <mitsharm@redhat.com>" \
  -m "feat: add <role_name> role skeleton"
```

## A5 — Offer to write tasks

Ask:
> "Role skeleton is ready. Would you like me to write the tasks now?
>
> 1. **Yes** — give me more detail about what the role should do and I'll write
>    `tasks/main.yml` and `defaults/main.yml`
> 2. **No** — leave the stubs as-is, I'll write the tasks myself"

**If 1 (Yes)** → follow `procedures/write-role-tasks.md`.
**If 2 (No)** → go to A6.

## A6 — Final Report

Print a structured report:

```
==================================================
 FINAL REPORT
==================================================

1. ROLES ADDED
   - <role_name>

2. FILES AND DIRECTORIES CREATED
   List every path written during A4 (and A5 if tasks were written):
   - automation/ansible/roles/<role_name>/              (directory)
   - automation/ansible/roles/<role_name>/tasks/        (directory)
   - automation/ansible/roles/<role_name>/defaults/     (directory)
   - automation/ansible/roles/<role_name>/meta/         (directory)
   - automation/ansible/roles/<role_name>/tasks/main.yml
   - automation/ansible/roles/<role_name>/defaults/main.yml
   - automation/ansible/roles/<role_name>/meta/main.yml
   - automation/ansible/roles/<role_name>/README.md
   (add any additional task files written during A5)

3. FILES AND DIRECTORIES MODIFIED
   List only files that already existed and were changed.
   If nothing was modified: "None — all files were newly created."

4. ERRORS
   List any errors that occurred (cp failures, git errors, YAML parse errors).
   If none: "None."

5. IDEMPOTENCY
   Re-running this skill with the same role name will be caught at step A3
   (duplicate check). Existing files will NOT be overwritten unless the user
   explicitly chooses to continue. The git commit is guarded by
   `git diff --cached --quiet` — no empty commits will be created.
==================================================
```

Ask: "Add another role?"

If yes → restart from A1. If no → STOP.
