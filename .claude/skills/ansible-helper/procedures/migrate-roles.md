# Import Roles from Git — Path B

Discovers all Ansible roles in a remote Git repository and copies selected ones
into `automation/ansible/roles/`.

## B1 — Ask for Git URL

Ask:
> "Paste the Git URL of the repository containing the role(s) you want to import:"

Wait for URL.

## B2 — Clone the repo

```bash
WORK_DIR=$(mktemp -d)
git clone --depth=1 <git_url> "$WORK_DIR/source_repo" 2>&1
```

If clone fails → show the full error output. **STOP.**

Note the repo name from the URL (last path segment, strip `.git`) as `source_repo_name`.

## B3 — Detect repo type and enumerate roles

Inspect the clone to determine what it contains. Run these checks in order:

**Check A — Ansible Collection** (has `galaxy.yml` at repo root):
```bash
test -f "$WORK_DIR/source_repo/galaxy.yml" && echo "COLLECTION"
ls -1 "$WORK_DIR/source_repo/roles/" 2>/dev/null
```
If COLLECTION → roles are the immediate subdirectories of `$WORK_DIR/source_repo/roles/`.

**Check B — Single role** (has `tasks/main.yml` at repo root):
```bash
test -f "$WORK_DIR/source_repo/tasks/main.yml" && echo "SINGLE_ROLE"
```
If SINGLE_ROLE → the entire repo is one role. Infer role name from `source_repo_name`
by stripping `ansible-role-`, `role-`, `ansible_role_` prefixes and converting to snake_case.

**Check C — Multi-role monorepo** (neither A nor B):
```bash
find "$WORK_DIR/source_repo" -maxdepth 2 -name "main.yml" -path "*/tasks/main.yml" \
  | sed 's|/tasks/main.yml||' \
  | sed "s|$WORK_DIR/source_repo/||" \
  | sort
```
Each result is a candidate role path. Role name is the directory name (last segment).

## B4 — Present role list and ask for selection

Build the list of discovered role names from whichever check matched.

If the list is empty → show:
> "No Ansible roles were found in that repository. It may not follow the standard
> role layout (`tasks/main.yml`). Check the URL and try again."
**STOP.**

Show:
> "Found the following roles in **`<source_repo_name>`**:
>
>  1. `role_alpha`
>  2. `role_beta`
>  3. `role_gamma`
>  ...
>
> Which roles would you like to import?
> - Type `all` to import everything
> - Type role numbers separated by commas to select specific ones (e.g. `1,3`)"

Wait for the user's selection. Resolve it to a final list of role names.

## B5 — Duplicate check

For each selected role name:
```bash
test -d automation/ansible/roles/<role_name> && echo "EXISTS" || echo "NEW"
```

For each that EXISTS, ask:
> "`<role_name>` already exists at `automation/ansible/roles/<role_name>/`.
>
> 1. Overwrite
> 2. Skip
> 3. Rename"

Apply the user's choice before proceeding to copy.

## B6 — Copy roles

For each selected role (applying the choice from B5):

**Collection (Check A):**
```bash
cp -r "$WORK_DIR/source_repo/roles/<role_name>" "automation/ansible/roles/<role_name>"
```

**Single role (Check B):**
```bash
cp -r "$WORK_DIR/source_repo/." "automation/ansible/roles/<role_name>"
```

**Monorepo (Check C):**
```bash
cp -r "$WORK_DIR/source_repo/<role_name>" "automation/ansible/roles/<role_name>"
```

After all copies are done:
```bash
rm -rf "$WORK_DIR"
```

## B7 — Update meta/main.yml

For each copied role:
- If `automation/ansible/roles/<role_name>/meta/main.yml` exists → read it, set
  `galaxy_info.author` to `owner_email` from spec.yaml, leave all other fields intact.
- If `meta/main.yml` is missing → create it using the template in
  `references/collection-structure.md`, filling `<role_name>`, `<owner_email>`, and
  leaving `<role_description>` as a placeholder comment.

## B8 — Commit

```bash
git add automation/ansible/roles/
git diff --cached --quiet || git commit --author="Mitesh Sharma <mitsharm@redhat.com>" \
  -m "feat: import roles from <source_repo_name>"
```

## B9 — Final Report

Print a structured report:

```
==================================================
 FINAL REPORT
==================================================

1. ROLES ADDED
   List every role that was successfully copied:
   - <role_name_1>   (source: <source_repo_name>/roles/<role_name_1>)
   - <role_name_2>   (source: <source_repo_name>/roles/<role_name_2>)
   - ...
   Skipped (user chose skip at B5): <role_name> — already existed.

2. FILES AND DIRECTORIES CREATED
   For each imported role list its top-level directory and any new files
   written during B6 and B7 (e.g. meta/main.yml stubs created for roles
   that were missing one). Example:
   - automation/ansible/roles/<role_name_1>/        (directory + all contents)
   - automation/ansible/roles/<role_name_1>/meta/main.yml   (created — was missing)

3. FILES AND DIRECTORIES MODIFIED
   List only files that already existed and were changed (e.g. meta/main.yml
   where only the author field was updated).
   If nothing was modified: "None — all files were newly created."

4. ERRORS
   List any errors that occurred (clone failures, cp cycle warnings, YAML
   parse errors, git errors).
   If none: "None."

5. IDEMPOTENCY
   Re-running this skill with the same source repo will re-present the same
   role list. Any role that already exists in automation/ansible/roles/ will
   be flagged at step B5 — the user must explicitly choose overwrite, skip,
   or rename. The git commit is guarded by `git diff --cached --quiet` — no
   empty commits will be created.
==================================================
```

Ask: "Import more roles from another repository?"

If yes → restart from B1. If no → STOP.
