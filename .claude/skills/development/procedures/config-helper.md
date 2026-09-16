# Showroom Config Helper

You help authors set up and configure Showroom content repositories. You handle the
first decision (showroom vs zerotouch content mode), generate the correct config files,
configure tabs, and create automation directory skeletons.

See @rhdp-publishing-house/skills/development/references/showroom-patterns.md for the three lab patterns and their complete config examples.
See @rhdp-publishing-house/skills/development/references/config-files.md for the full reference on every config file.
See @rhdp-publishing-house/skills/development/references/gitops-patterns.md for GitOps (Helm + ArgoCD) conventions and patterns.
See @rhdp-publishing-house/skills/development/references/ansible-conventions.md for Ansible collection naming rules and how to fill in the starter collection.

## Step 1 — Detect Repo Context

Check these files and directories silently to understand the current state:

| Check | Meaning |
|-------|---------|
| `publishing-house/spec.yaml` exists | This is a Publishing House project |
| `.scaffolds/` directory exists | PH project that has NOT been scaffolded yet |
| `site.yml` exists | Antora playbook already configured |
| `ui-config.yml` exists | Showroom UI already configured |
| `content/antora.yml` exists | Antora component present |
| `runtime-automation/` exists | Zerotouch automation skeleton in place |
| `config/` directory exists | ZT Guided infrastructure config present |

## Step 2 — Route Based on State

### Route A: PH project with `.scaffolds/` still present

The project has not been scaffolded. Read the intake spec to determine the pattern, then delegate to `scaffold.py`.

**A.1 — Read spec.yaml to determine pattern:**

Read `publishing-house/spec.yaml` and extract `project.showroom_type`. Map to scaffold pattern:

| `project.showroom_type` | scaffold.py `--pattern` |
|--------------------------|-------------------------|
| `classic` | `agd-open` |
| `zero_touch` | `zt-guided` |
| `guided` | `agd-guided` (not currently offered during intake) |

If `project.showroom_type` is empty, unset, or unrecognised, fall back to asking:
> I couldn't determine the showroom type from your spec. Which lab pattern do you want?
>
> 1. **Open** (`agd-open`) — self-paced, free-form navigation
> 2. **Guided** (`agd-guided`) — sequential modules with solve/validate buttons
> 3. **ZT Guided** (`zt-guided`) — guided + Project Zero infrastructure

Also read `project.intake_type` (`new` or `migration`). This determines whether **A.1b** runs
before scaffolding, and whether `--migration` is passed to `scaffold.py` in **A.2**.

**A.1b — Align migrated content naming with intake modules (migration only):**

Only runs when `project.intake_type` is `migration` **and** the pattern resolved above is
`zt-guided`. Skip entirely otherwise — fresh (`new`) intake projects have no content yet at this
point (pages are created lazily during development, already using the right names), and no other
pattern has a `-migration` scaffold overlay today.

The `rhdp-publishing-house-skills:migrate` skill's Phase 2 (see `migrate/procedures/02-module-outlines-from-content.md`)
already renames `content/modules/ROOT/pages/*.adoc`, `runtime-automation/*/` folders, and
`nav.adoc` xrefs to the canonical `module-NN-<slug>` names *during intake*, right when the
outlines are generated. **It does not touch `ui-config.yml`** — that file isn't read or written
by the migrate skill at all. So by the time this step runs, `ui-config.yml`'s `antora.modules`
list is normally the *only* thing still using the pre-migration names. Fix that, and only fall
back to renaming files/folders yourself for anything migrate's Phase 2 missed (e.g. repos
migrated before that step existed, or a page added after intake completed):

1. Read the existing `ui-config.yml`'s `antora.modules` list, in order, **excluding** the
   `index` entry — this is the pre-migration module stem order.
2. List `publishing-house/spec/modules/module-*.md`, sorted numerically, and take each file's
   stem (`module-01-<slug>`, `module-02-<slug>`, ...) — one per `spec.modules[]` entry, in the
   same order.
3. Zip the two lists positionally. **If the counts don't match, STOP** — show the author both
   lists and ask how to proceed rather than guessing:
   > The migrated content has `<N>` modules (`<old names>`) but intake generated `<M>` module
   > outlines (`<new names>`). I can't safely auto-align these — how would you like to map them?
4. For each `(old, new)` pair where `old != new`:
   - **Always** update that entry's `name:` field in `ui-config.yml` to `<new>` — this is the
     one thing intake never does. Also offer to sync `label:` to the matching outline's title
     from `spec.modules[]`, but only with author confirmation — never silently rewrite a label
     the author may have already tuned.
   - **Only if** `content/modules/ROOT/pages/<old>.adoc` still exists (migrate's Phase 2 didn't
     already rename it): `git mv` it to `<new>.adoc` and update its `xref:` target in
     `content/modules/ROOT/nav.adoc`. If `<new>.adoc` already exists instead, skip — it's already
     aligned.
   - **Only if** `runtime-automation/<old>/` still exists: `git mv` it to
     `runtime-automation/<new>/`. If `runtime-automation/<new>/` already exists instead, skip.
5. Commit:
   ```bash
   git add -A
   git commit -m "chore: align migrated module naming with intake outlines"
   ```
   Skip the commit if nothing changed (e.g. migrate's Phase 2 already handled everything and
   `ui-config.yml` was already correct too).

**A.2 — Confirm and run scaffold.py:**

Read everything needed to build the full scaffold plan before showing anything to the user:

- `project.showroom_type` (already mapped to a pattern in A.1)
- `project.intake_type` (already read in A.1) — if `migration`, pass `--migration`. In migration
  mode, `scaffold.py` never overwrites existing content. `runtime-automation/`,
  `setup-automation/`, and `config/` are fully hands-off (never wiped, never filled in — the
  migrated repo's shell-script-based automation there is structured completely differently
  from the fresh scaffold's ansible-playbook-per-module stubs). Other pattern files still get
  genuinely-missing files filled in, and the migration-specific `qa-automation/` is overlaid on
  top. Requires A.1b to have already run so the existing content is aligned to the right names
  first.
- `project.automation_type` from `publishing-house/spec.yaml`. This is the only chance to scaffold
  automation directories automatically — `scaffold.py` deletes `.scaffolds/` (including
  `.scaffolds/automation/`) once it completes, so automation scaffolding must happen in this same
  invocation, not as a later separate step. See the Automation Scaffolding reference (below) for what
  each value creates. If `automation_type` is empty, unset, or unrecognised, omit `--automation`
  entirely.
- `spec.environment.topology` — if it already happens to be `shared-cluster` at this point, also pass
  `--topology shared-cluster` (usually it won't be known yet — topology is normally decided later
  during intake).

Show the author the **entire** plan in a single confirmation, then proceed automatically — never ask
whether they'd rather scaffold it themselves; that's what this step exists to do for them:

> Based on your spec, I'll scaffold this project as:
> - **Pattern:** `showroom_type: <value>` → **<pattern description>** (`<pattern-name>`)
> - **Automation:** <"none" | "`<automation_type>` → `automation/<...>/`" (+ `bootstrap-tenant/` if topology is `shared-cluster`)>
> [If migration:] - **Migration:** existing `runtime-automation/`, `setup-automation/`, and `config/` are preserved as-is, untouched — `ui-config.yml` and other pattern files only have genuinely missing files filled in, and `qa-automation/` is replaced with the migration-aware version
>
> Proceed?

Then run:
```bash
python scaffold.py --pattern <pattern-name> --automation <automation_type> --force
```
(Omit `--automation` if `automation_type` was empty/unset. Add `--topology shared-cluster` if
applicable. Add `--migration` if `project.intake_type` is `migration`.)

**A.3 — Configure tabs from spec:**

After scaffolding, the `ui-config.yml` has placeholder tabs. Follow the Tab Advisor procedure (below) to replace them with real tabs based on the spec's infrastructure.

**A.4 — Automation directories already scaffolded:**

`scaffold.py` (step A.2) already created `automation/` if `--automation` was passed. See the
Automation Scaffolding reference (below) for a summary of what was created, and for how to add
`automation/gitops/bootstrap-tenant/` manually later if topology turns out to be `shared-cluster`
after `.scaffolds/` is already gone.

**A.5 — Verify and adjust:**

After tab and automation configuration, proceed to Route C (modification flow) if the user wants further changes.

### Route B: No config files exist (new repo)

No `site.yml` or `ui-config.yml` found. Run the full setup flow.

**B.1 — Choose content mode:**

> First decision: how should learners navigate your lab?
>
> 1. **Showroom** (`type: showroom`) — free-form browse with table of contents. Best for self-paced workshops, demos, and reference environments.
> 2. **Zerotouch** (`type: zerotouch`) — guided linear progression with Next/Previous, Solve, and Validate buttons. Best for instructor-led labs requiring sequential completion.

**B.2 — Choose infrastructure type:**

> What infrastructure will your lab run on?
>
> 1. **OCP** — OpenShift cluster (AgnosticD v2)
> 2. **VM** — Virtual machines with bastion (AgnosticD v2)
> 3. **ZT** — Project Zero (zero-touch VMs with custom networking)

ZT always uses zerotouch content mode. If the user chose showroom mode and ZT infrastructure, confirm:
> ZT infrastructure typically uses zerotouch mode for solve/validate automation. Continue with showroom mode, or switch to zerotouch?

**B.3 — Ask for lab title:**

> What is the title for your lab?

**B.4 — Generate config files:**

Based on the content mode and infrastructure type, determine the pattern (Open, Guided, or ZT Guided) and generate all files. Use the exact templates from @rhdp-publishing-house/skills/development/references/showroom-patterns.md — do not improvise config structure.

Files to generate:

1. **`site.yml`** — set title, choose the correct theme bundle, register extensions. For zerotouch, include the dev-mode extension.

2. **`ui-config.yml`** — use the correct format for the content mode:
   - Showroom: `type: showroom` with `view_switcher` and `tabs`
   - Zerotouch: `antora:` block with module labels and `tabs`
   - Configure tabs using the Tab Advisor procedure (below)

3. **`content/antora.yml`** — set title, nav path, default attributes for the infrastructure type

4. **`content/modules/ROOT/nav.adoc`** — create with index.adoc entry

5. **`content/modules/ROOT/pages/index.adoc`** — create a minimal stub

6. **`podman-compose.yaml`** — create from the standard template in the config-files reference

7. If zerotouch:
   - **`runtime-automation/module-01/`** — create `setup.yml`, `solve.yml`, `validate.yml` stubs

8. If ZT Guided:
   - **`config/instances.yaml`** — default RHEL 9.5 bastion VM
   - **`config/networks.yaml`** — default + secondary networks
   - **`config/firewall.yaml`** — egress TCP 443
   - **`setup-automation/setup.yml`** — environment setup stub

**B.5 — Summarize what was created:**

Show a tree of generated files and explain the key next steps:
- Edit `content/antora.yml` to set your lab title and runtime attribute defaults
- Replace placeholder tabs in `ui-config.yml` with real service URLs
- Start writing content in `content/modules/ROOT/pages/`
- Preview locally with `podman-compose up`

### Route C: Config files exist (modification flow)

The repo already has `site.yml` and/or `ui-config.yml`.

1. Read existing config files to detect the current pattern. Use the detection rules from @rhdp-publishing-house/skills/development/references/showroom-patterns.md.

2. Understand what the user wants to change. Common requests:
   - **Add a tab** — add entry to `tabs:` list in ui-config.yml
   - **Remove a tab** — remove entry from `tabs:` list
   - **Change tab order** — reorder entries
   - **Configure tabs from spec** — read `spec.environment` and run the Tab Advisor procedure
   - **Adjust layout width** — change `default_width` (showroom mode only)
   - **Change title** — update `site.title` in site.yml and/or `title` in antora.yml
   - **Add antora.yml attributes** — add to `asciidoc.attributes`
   - **Switch content mode** — change theme in site.yml AND format of ui-config.yml (significant change — confirm with user)
   - **Add a module** — create page file, update nav.adoc, and if zerotouch update `antora.modules` in ui-config.yml and create runtime-automation stubs
   - **Replace placeholders** — swap `/placeholder` tabs with real URLs
   - **Set up automation** — see the Automation Scaffolding reference (below). If `.scaffolds/`
     still exists, delegate to Route A instead so `scaffold.py --automation` can run before it's
     removed. If `.scaffolds/` is already gone, automation directories must be created manually
     (pull them from the `rhdp-publishing-house-template` repo directly) — `scaffold.py --automation`
     only works on a project's first scaffolding run

3. Apply changes while preserving format consistency. When modifying ui-config.yml, maintain the correct format for the detected content mode.

4. After modifications, suggest running `rhdp-publishing-house:config-reviewer` to validate the result.

### Fixing J-rule findings from config-reviewer

When the author asks for help resolving J-02 through J-05 findings from `config-reviewer` (PH projects only):

**J-02 — outline/page filename mismatch:**
1. Present the mismatched pair: the outline filename (`publishing-house/spec/modules/<stem>.md`) and
   whatever `.adoc` file(s) exist in `content/modules/ROOT/pages/` that don't match its stem.
2. Ask the author how to resolve it — never rename files without confirmation:
   > I found `<outline-stem>.md` with no matching page. Did you mean `<closest-existing-page>.adoc`?
   > I can rename it to `<outline-stem>.adoc` and update its `nav.adoc` xref to match. Proceed?
3. If confirmed: rename the page file, then update its `xref:` entry in `content/modules/ROOT/nav.adoc`
   in the same step so the two never drift apart again.

**J-03 — index.adoc missing:**
Create the minimal stub (same template as Route B.5), then tell the author to fill in the real content
themselves or ask `writer-helper` to generate it.

**J-04 — conclusion.adoc missing:**
Do NOT create a stub — a placeholder conclusion isn't useful, since it needs the full list of completed
modules and learning objectives to be meaningful. Instead:
> `conclusion.adoc` isn't written yet. Once all your modules are marked complete, ask **writer-helper**
> to "generate the conclusion" — it needs the full module list to build the "What You've Learned" recap.

**J-05 — nav.adoc missing:**
Create `nav.adoc` with an index entry (same template as Route B.4), then scan
`content/modules/ROOT/pages/` for any existing `.adoc` files and offer to append `xref:` entries for
them too:
> I created `nav.adoc` with an index entry. I also found `<N>` existing page(s) not yet listed —
> want me to add them?

## Tab Advisor

This procedure is used by Routes A, B, and C to configure tabs based on the lab's infrastructure. For PH projects, read `publishing-house/spec.yaml`. For standalone repos, use the infrastructure type from Step 1 detection or ask the user.

### Read the spec

Extract from `publishing-house/spec.yaml` (if present):
- `spec.environment.platform` — `ocp` or `rhel-vms`
- `spec.environment.vms_per_student` — VM roles (for `rhel-vms` platform)
- `publishing-house/spec/design.md` Products section — hints at deployed applications

### For `platform: ocp`

**OCP Console** — suggest by default:
```yaml
- name: OCP Console
  url: 'https://console-openshift-console.${DOMAIN}'
```

**Terminal** — ask the user which type they need:
> Your lab runs on OCP. Which terminal access do learners need?
>
> 1. **Bastion terminal** — SSH to a bastion host via wetty (requires a bastion VM in provisioning)
> 2. **OCP Terminal** — browser-based terminal with `oc` CLI pre-configured (no bastion required)
> 3. **Both** — stacked vertically in one tab
> 4. **None** — no terminal tab needed

Bastion terminal:
```yaml
- name: ">_ terminal"
  path: /wetty
  port: 443
```

OCP Terminal:
```yaml
- name: ">_ OCP Terminal"
  url: 'https://codeserver-codeserver.${DOMAIN}'
```

Both (stacked):
```yaml
- name: ">_ Terminals"
  path: /wetty
  port: 443
  secondary_name: OCP Terminal
  secondary_path: /codeserver
  secondary_port: 443
```

**Application tabs** — ask about deployed services:
> Will any applications be deployed to OCP that learners need to access in the UI?
> For example: ArgoCD, RHACS, Grafana, Developer Hub, custom app consoles.
>
> If you know the route names now, I can add tabs. Otherwise I can add placeholder tabs we'll fill in during development.

For each app, generate:
```yaml
- name: <App Name>
  url: 'https://<route-name>-<namespace>.${DOMAIN}'
```

Common OCP app routes:
- ArgoCD: `openshift-gitops-server-openshift-gitops`
- RHACS: `central-stackrox`
- Developer Hub: `backstage-developer-hub-backstage`
- Grafana: `grafana-grafana`

If the user doesn't know routes yet, add a placeholder:
```yaml
- name: <App Name>
  url: /placeholder
```

### For `platform: rhel-vms`

Read `spec.environment.vms_per_student` to understand the VM roles.

**Terminal tabs** — for each VM role, suggest a terminal tab. Ask whether to use separate tabs or stacked (vertically split) terminals:

> Your lab provisions these VMs per student:
> [list roles from vms_per_student]
>
> How should terminal access be laid out?
> 1. **Separate tabs** — one tab per VM
> 2. **Stacked** — two terminals vertically split in one tab (good for watch + work)
> 3. **Single** — just the bastion, other VMs accessed via SSH from there

Separate tabs example:
```yaml
- name: ">_ Bastion"
  path: /wetty
  port: 443
- name: ">_ Worker"
  path: /terminal2
  port: 443
```

Stacked example:
```yaml
- name: ">_ Terminals"
  path: /wetty
  port: 443
  secondary_name: Worker
  secondary_path: /terminal2
  secondary_port: 443
```

### Vertical split (stacked terminals)

Tabs can be vertically split into top and bottom panels using `secondary_*` properties. This is commonly used to:
- Run a `watch` command in one terminal while working in the other
- Monitor logs while executing steps
- SSH to different hosts simultaneously

Offer split when multiple terminals or VMs are involved. The user can combine any two services in a single split tab.

### Documentation and other links

Always ask:
> Would you like to add any documentation or reference links as tabs?
> For example: product docs, API references, architecture diagrams.

External URL tabs:
```yaml
- name: Product Docs
  url: 'https://docs.redhat.com/...'
```

Sites that set `X-Frame-Options` or `Content-Security-Policy` headers blocking iframes will show a blank pane in the tab.

### Write tabs to ui-config.yml

After the conversation, write the complete `tabs:` list to `ui-config.yml`, replacing any placeholder entries from scaffolding. Preserve the rest of the file (type, view_switcher, antora block).

### Port note

Every `path`/`secondary_path` tab should set an explicit `port`/`secondary_port` — typically `443` for standard HTTPS — for backward compatibility with older/production Showroom UI builds that don't reliably support omitting it. Only use a different value when the service genuinely runs on a non-standard port. `url` tabs never need `port` — the URL is already fully specified.

## Automation Scaffolding

`scaffold.py --automation {ansible,gitops,both}` creates the automation directory skeleton —
this is handled by the script itself (Route A, step A.2), not by manually copying files. Its
source templates live in `.scaffolds/automation/`, which only exists before the project's first
scaffolding run.

### What each automation_type creates

Reading `project.automation_type` from `publishing-house/spec.yaml`:

| `automation_type` | Flag | Creates |
|---|---|---|
| `ansible` | `--automation ansible` | `automation/ansible/` |
| `gitops` | `--automation gitops` | `automation/gitops/bootstrap-infra/` (+ `automation/gitops/bootstrap-tenant/` if `--topology shared-cluster` was also passed) |
| `both` | `--automation both` | all of the above |

If `automation_type` is empty, unset, or unrecognised → omit `--automation` and skip silently.

If the target directory already exists for a given type (`automation/gitops/` or `automation/ansible/`),
`scaffold.py` clears and recreates it (with `--force`) or prompts before overwriting — see its
`--help` for the exact behavior.

- **`automation/gitops/bootstrap-infra/`** — minimal Helm chart with a single test namespace that
  proves the ArgoCD Application deploys correctly. The author replaces this with real workloads
  during development.
- **`automation/gitops/bootstrap-tenant/`** — per-user tenant chart with a single namespace and
  edit RoleBinding. The deployer creates one ArgoCD Application per user, injecting `username` and
  `deployer.domain`. Only created when `spec.environment.topology` is `shared-cluster` — for
  `per-student` or `cnv-pool` topologies, each student gets their own cluster, so there is no
  multi-tenant deployment and this should NOT be created.
- **`automation/ansible/`** — starter Ansible collection. Its `galaxy.yml`, example role, and
  READMEs still have `<placeholder>` tokens after scaffolding — fill them in immediately after
  (see below) rather than leaving them for the author to find.

### Filling in the Ansible collection

Immediately after `scaffold.py` creates `automation/ansible/` (whether from `--automation ansible`
or `--automation both`), fill in its placeholders using real project values. Full rules and
rationale are in @rhdp-publishing-house/skills/development/references/ansible-conventions.md —
follow it exactly rather than improvising the derivation or the naming validation.

1. **Read project identity.** From `catalog-info.yaml`: `metadata.name` (repo name), annotation
   `ph.rhdp.io/github-user`, annotation `ph.rhdp.io/owner` (email). Fall back to
   `publishing-house/spec.yaml` (`project.slug`, `project.owner_email`) for whichever fields
   `catalog-info.yaml` doesn't have. If neither source has a value, ask the author directly —
   never write a placeholder into a real file.

2. **Derive and validate the namespace.** Replace `-` with `_` in the repo name. If the result
   starts with a digit, is under 3 characters, or has consecutive underscores, it fails Ansible's
   naming rules — stop and ask the author for a valid namespace instead of writing a broken one:
   > The repo name `<repo-name>` doesn't map to a valid Ansible namespace (`<reason>`). What
   > namespace would you like to use? Lowercase letters/digits/underscores only, 3+ characters,
   > can't start with a digit or underscore, no double underscores.

3. **Fill `automation/ansible/galaxy.yml`:**
   - `namespace: "<derived-or-provided>"`
   - `authors: ["<github-user> <owner-email>"]`
   - `repository: "https://github.com/rhpds/<repo-name>"`
   - Leave `name` alone — it already ships as `"automation"`, not a placeholder. Leave `license`
     untouched too — there's no signal for which license the org wants.

4. **Fill `automation/ansible/roles/example/meta/main.yml`** — set `galaxy_info.author` to the
   GitHub username. Leave `license` untouched, same reasoning.

5. **Update the FQCN examples** in `automation/ansible/README.md` and
   `automation/ansible/roles/example/README.md` — both ship with `<your_namespace>.automation.*`
   placeholders; replace `<your_namespace>` with the derived namespace in each.

Include the resulting namespace in the overall scaffolding summary (below), not as a separate message.

### Adding bootstrap-tenant/ after the fact

Topology is normally decided during intake, *after* the first `scaffold.py` run — by which point
`.scaffolds/automation/` is already gone. If topology later turns out to be `shared-cluster` and
`bootstrap-tenant/` wasn't created on that first run, it must be added manually: pull
`.scaffolds/automation/gitops/bootstrap-tenant/` from the `rhdp-publishing-house-template` repository
directly and copy it into this project's `automation/gitops/bootstrap-tenant/`. There is no
automated re-scaffold path once `.scaffolds/` has been removed.

### After scaffolding

Commit the automation directories:
```bash
git add automation/
git commit -m "feat: scaffold automation directories"
```

Summarise what was created:
> **Automation skeleton created:**
> - `automation/gitops/bootstrap-infra/` — Helm chart with a test namespace (replace with real workloads)
> [If tenant:] - `automation/gitops/bootstrap-tenant/` — per-user namespace and RBAC
> [If ansible:] - `automation/ansible/` — namespace `<namespace>`, collection `automation`,
>   author `<github-user> <owner-email>`. Still needs: pick a `license`, then rename or replace
>   `roles/example/` with real automation.
>
> See the [GitOps patterns reference](references/gitops-patterns.md) for sync-wave ordering,
> operator quirks, and deployment conventions.

## Rules

- Always use the exact config structures from the reference files — do not invent new YAML keys
- When generating zerotouch ui-config.yml, the `antora.modules` entries must match the page filenames in `content/modules/ROOT/pages/`
- Tab URLs that reference cluster services should use `${DOMAIN}` variable substitution (not `{DOMAIN}`)
- Built-in variables available in ui-config.yml: `${DOMAIN}`, `${GUID}`, `${USER}`
- Custom variables can be defined in `content/antora.yml` under `asciidoc.attributes.environment_variables` — these are exported before `envsubst` runs on ui-config.yml at deploy time
- When suggesting tab URLs that need custom values, either use a built-in variable or add the variable to `environment_variables` in antora.yml
- Never modify `site.yml` content source settings — `start_path: content` and `url: .` are fixed
- For PH projects, do not modify `publishing-house/` directory contents — that is PH skill territory
- When switching content modes, both `site.yml` (bundle URL) and `ui-config.yml` (format) must change together
- When fixing J-02 findings, never rename a page file or edit its `nav.adoc` xref without author confirmation
- Never stub `conclusion.adoc` — redirect to `writer-helper` instead, since it needs full module context
