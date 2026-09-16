# Validation Rules Reference

All validation rules enforced by the `showroom:config-reviewer` skill.

## Severity Levels

| Level | Meaning |
|-------|---------|
| **CRITICAL** | Deployment will fail or produce a broken UI. Must fix before deploying. |
| **HIGH** | Likely to cause visible issues at deploy time. Should fix. |
| **MEDIUM** | Best practice violation. May cause subtle issues or developer confusion. |
| **LOW** | Style or convention recommendation. Optional to fix. |

---

## site.yml Rules (S-rules)

### S-1 — Bundle URL must be a known valid theme

- **Severity:** CRITICAL
- **Auto-fix:** No — requires author to confirm intended theme

Valid URLs contain `rhdp_showroom_theme` or `nookbag-bundle` in the GitHub releases path. Unknown bundle URLs may indicate a typo or outdated reference.

### S-2 — start_page must match the antora.yml component name

- **Severity:** HIGH
- **Auto-fix:** Yes — read antora.yml `name` field and correct `start_page`

Format: `<component-name>::index.adoc`. If antora.yml has `name: modules`, start_page must be `modules::index.adoc`.

### S-3 — Content source start_path must be `content`

- **Severity:** HIGH
- **Auto-fix:** Yes — set `start_path: content`

The Antora component descriptor (`antora.yml`) lives in the `content/` directory.

### S-4 — Mermaid and tabs Antora extensions should be registered

- **Severity:** MEDIUM
- **Auto-fix:** Yes — add missing extension blocks (Open and Guided patterns only)
- **Skip entirely for ZT Guided** — do NOT flag as a finding and do NOT offer to auto-fix.
  Project Zero infra doesn't support these extensions yet, so `zt-guided` scaffolding
  intentionally omits them. Adding them would produce a broken build for the author, not a fix.

Check for `@sntke/antora-mermaid-extension` and `@andrew-jones/antora-tabs-extension` in
`antora.extensions` — only for Open and Guided patterns (detected per
`showroom-patterns.md` Step 2). If the detected pattern is ZT Guided, skip this rule
entirely rather than reporting it as PASSED or FAILED.

### S-5 — Output directory should be `./www`

- **Severity:** MEDIUM
- **Auto-fix:** Yes — set `output.dir: ./www`

The `podman-compose.yaml` and Showroom container expect output in `./www`.

### S-6 — Site title should not be a placeholder

- **Severity:** LOW
- **Auto-fix:** No — requires author input

Check that `site.title` is not `My Project`, `Showroom Template Demo`, or similar default values.

### S-7 — rhdp_showroom_theme: navbar_logo must be a known branding value

- **Severity:** LOW
- **Auto-fix:** No — informational

If `site.keys.navbar_logo` is set, validate it is a known branding value (e.g. `summit`, `rhdp`). Unknown values will fall back to default branding silently.

### S-8 — nookbag-bundle: dev-mode extension should be registered

- **Severity:** MEDIUM
- **Auto-fix:** Yes — add dev-mode extension block

The extension must be registered as `- require: /antora/lib/dev-mode.js` with `enabled: true`. This is a zerotouch convention — the dev-mode extension is provided by the Showroom container at build time and is not available during local Antora CLI builds.

---

## ui-config.yml Rules (U-rules)

### U-1 — Format must match the content mode

- **Severity:** CRITICAL
- **Auto-fix:** No — changing format requires restructuring the entire file

Open/showroom pattern uses `type: showroom` header. Zerotouch/guided pattern uses `antora:` block with module labels. If site.yml uses `rhdp_showroom_theme`, ui-config must have `type: showroom`. If site.yml uses `nookbag-bundle`, ui-config must have `antora:` block.

### U-2 — At least one tab must be defined

- **Severity:** HIGH
- **Auto-fix:** No — requires author to specify which services to expose

A showroom without tabs renders a blank right pane.

### U-3 — Every tab must have a name property

- **Severity:** HIGH
- **Auto-fix:** No — requires author to provide meaningful names

Tabs without names appear as blank tab headers.

### U-4 — Each tab must have either url or path (not both, not neither)

- **Severity:** HIGH
- **Auto-fix:** Partial — can warn about the issue but needs author to decide the correct value

Using both is ambiguous. Using neither produces a broken tab.

### U-4b — path-based tabs should include an explicit port for backward compatibility

- **Severity:** MEDIUM
- **Auto-fix:** Yes — add `port: 443` (or `secondary_port: 443`) when missing

Every `path`/`secondary_path` tab should set an explicit `port`/`secondary_port` — typically `443` — even though 443 is technically the default. Older/production Showroom UI builds don't reliably support omitting it, so relying on the default causes tabs (e.g. `/wetty` terminals) to fail to load in some environments. Only use a different value when the service genuinely runs on a non-standard port. `url` tabs are exempt — the URL is already fully specified and doesn't use this mechanism.

### U-5 — Placeholder tabs should be replaced before deployment

- **Severity:** LOW
- **Auto-fix:** No — requires author to provide real URLs

Placeholder tabs (`url: /placeholder`) render a stub panel for local development only.

### U-6 — Variable substitution must use `${VARIABLE}` syntax and reference known variables

- **Severity:** MEDIUM
- **Auto-fix:** Yes — add `$` prefix to bare `{VARIABLE}` references

The `${}` wrapper is required in ui-config.yml. AsciiDoc attribute substitution uses `{attribute}` without `$`, but ui-config.yml is different. Common mistake: `{DOMAIN}` instead of `${DOMAIN}`.

**Built-in variables** (always valid): `${DOMAIN}`, `${GUID}`, `${USER}`

**Custom variables:** Any key defined in `content/antora.yml` under `asciidoc.attributes.environment_variables` is also valid. Check this map when encountering non-built-in `${VAR}` references — if the variable is listed there, it passes. If not, flag as a warning (the variable may resolve to an unintended container environment variable or be empty at runtime).

### U-6b — Non-built-in variables in tab URLs should be defined in antora.yml environment_variables for local development

- **Severity:** LOW
- **Auto-fix:** No — requires author to either add the variable to antora.yml or confirm it's injected by the deployment

If a tab URL contains `${VAR}` where VAR is not one of `DOMAIN`, `GUID`, `USER` and is not listed in `content/antora.yml` `asciidoc.attributes.environment_variables`, flag it as a recommendation. In production the variable may be injected by the Helm chart or AgnosticV workload, but defining it in `environment_variables` ensures local preview (`podman-compose up`) works correctly. This is a development convenience, not a deployment requirement.

### U-7 — Zerotouch: antora.modules entries must match page filenames

- **Severity:** HIGH
- **Auto-fix:** Partial — can add missing entries, but label text needs author input

Each module `name` field must correspond to a file `<name>.adoc` in `content/modules/ROOT/pages/`. Missing entries mean Nookbag won't show that module in the progress bar. Extra entries with no matching file produce broken navigation.

---

## antora.yml Rules (A-rules)

### A-1 — name must be set

- **Severity:** HIGH
- **Auto-fix:** Yes — set to `modules` if missing

The component name (typically `modules`) is referenced by site.yml `start_page` and ui-config.yml `antora.name`.

### A-2 — title should not be a placeholder

- **Severity:** MEDIUM
- **Auto-fix:** No — requires author input

Check that it is not `My Lab Title`, `Your Lab Title`, or similar defaults.

### A-3 — nav path must be correct

- **Severity:** HIGH
- **Auto-fix:** Yes — set correct nav path

Should contain `- modules/ROOT/nav.adoc` pointing to the navigation file.

### A-4 — Common runtime attributes should have sensible defaults

- **Severity:** MEDIUM
- **Auto-fix:** Yes — add missing attributes with standard defaults

Check that `guid`, `ssh_user`, `ssh_password` (and `openshift_console_url` for OCP labs) are defined with default values. These are overridden by `user_data` at deploy time but are needed for local preview.

### A-5 — experimental: true should be set

- **Severity:** MEDIUM
- **Auto-fix:** Yes — add `experimental: true` to attributes

Enables the AsciiDoc UI macros: `btn:[]` (buttons), `kbd:[]` (keyboard shortcuts), `menu:[]` (menu paths). Without it, these macros render as literal text.

### A-6 — page-pagination: true should be set

- **Severity:** LOW
- **Auto-fix:** Yes — add `page-pagination: true` to attributes

Enables next/prev page links at the bottom of each page. For zerotouch this is less critical (Nookbag handles navigation) but still useful.

---

## nav.adoc Rules (N-rules)

### N-1 — All pages should be listed in nav.adoc

- **Severity:** MEDIUM
- **Auto-fix:** Partial — can suggest xref entries, but position and display name need author input

All pages in `content/modules/ROOT/pages/` should be listed in nav.adoc. Unlisted pages are accessible by direct URL but won't appear in the sidebar (unless dev-mode is enabled). This is a warning — some pages may be intentionally unlisted.

### N-2 — All xref targets in nav.adoc must exist as files

- **Severity:** HIGH
- **Auto-fix:** No — the file needs to be created or the xref removed; both require author decision

A broken xref produces a build warning and a dead link in the sidebar.

### N-3 — index.adoc should be the first entry

- **Severity:** LOW
- **Auto-fix:** Yes — reorder nav entries to put index first

Convention — the index page is the landing page and should appear first in navigation.

---

## Cross-file Rules (X-rules)

### X-1 — site.yml start_page component must match antora.yml name

- **Severity:** HIGH
- **Auto-fix:** Yes — update site.yml `start_page` to match antora.yml `name`

If antora.yml has `name: modules`, site.yml must have `start_page: modules::index.adoc`. A mismatch causes Antora to fail to resolve the start page.

### X-2 — Zerotouch: ui-config module names must match page filenames

- **Severity:** HIGH
- **Auto-fix:** Partial — can add missing module entries, label text needs author

`antora.modules` names in ui-config.yml must match page filenames in `content/modules/ROOT/pages/`. Each module with `name: foo` must have a corresponding `foo.adoc` file. Pages that exist but aren't listed in modules produce a warning (not error).

### X-3 — Zerotouch: runtime-automation must exist per module

- **Severity:** HIGH
- **Auto-fix:** Yes — can create stub playbook files

`runtime-automation/` must exist with solve and validate automation per module (except index). Each module listed in `antora.modules` (other than the index/intro page) should have a corresponding directory in `runtime-automation/<module-name>/` containing solve and validate files.

**Accepted file formats** — check for any of these per module directory:
- `.yml`/`.yaml` playbooks: `solve.yml`, `validate.yml` (or `validation.yml`), `setup.yml`
- `.sh` shell scripts: `solve-*.sh`, `validate-*.sh` (or `validation-*.sh`), `setup-*.sh`

A module directory with at least one solve file AND one validate file passes. Module-01 may also have a setup file. Do not flag a module for missing `.yml` if `.sh` files are present (or vice versa).

### X-4 — Theme and content mode must be consistent

- **Severity:** CRITICAL
- **Auto-fix:** No — requires author to decide which content mode they want, then both files need updating

`rhdp_showroom_theme` in site.yml requires `type: showroom` in ui-config.yml. `nookbag-bundle` in site.yml requires zerotouch format (`antora:` block) in ui-config.yml. A mismatch produces duplicate navigation (both Antora theme and Nookbag provide nav controls) or missing navigation (neither provides it).

### X-5 — ZT Guided: config and setup-automation directories must exist

- **Severity:** MEDIUM
- **Auto-fix:** Yes — can create stub config and setup files

`config/` directory must exist with `instances.yaml`, `networks.yaml`, and `firewall.yaml`. `setup-automation/` must exist with `setup.yml`. These are required for Project Zero infrastructure provisioning.

### X-6 — Terminal tabs must use path + port syntax, not url

- **Severity:** MEDIUM
- **Auto-fix:** Yes — convert `url: /wetty` to `path: /wetty` + `port: 443`

All patterns (AgD Open/Guided and ZT Guided alike) use `path: /wetty` + `port: 443` for terminal tabs — there is no longer an infra-specific variant. A tab with `url: /wetty` (instead of `path`) is outdated and should be converted; see U-4b for the port requirement itself.

---

## Ansible Collection Rules (G-rules) — Skip if `automation/ansible/galaxy.yml` does not exist

These rules validate the starter Ansible collection created by `scaffold.py --automation ansible`
(or `both`) and filled in by `config-helper`'s Filling in the Ansible Collection procedure. See
@rhdp-publishing-house/skills/development/references/ansible-conventions.md for the full naming
rules and derivation algorithm — these checks just verify the result matches it.

### G-1 — galaxy.yml namespace/authors must not be placeholders

- **Severity:** HIGH
- **Auto-fix:** Yes — re-run the config-helper fill-in procedure

Check `namespace` and `authors` in `automation/ansible/galaxy.yml` are not still
`<your_namespace>` or `<Your Name> <you@example.com>`. Placeholder values mean the collection was
scaffolded but never filled in. (`name` is not checked here — it ships as `"automation"` from the
template and is never a placeholder; see G-2 if it's been changed to something invalid.)

### G-2 — namespace and name must satisfy Ansible's collection naming rules

- **Severity:** CRITICAL
- **Auto-fix:** No — requires the author to choose a valid namespace

Both `namespace` and `name` must contain only lowercase alphanumeric characters and underscores, be
at least 3 characters long, not start with an underscore or a digit, and not contain consecutive
underscores. A violation means `ansible-galaxy collection install` will fail for anyone who tries
to consume the collection — this is not a style nit.

### G-3 — repository should point to the real repo, not a placeholder

- **Severity:** LOW
- **Auto-fix:** Yes — set to `https://github.com/rhpds/<repo-name>`

Check `repository` in `galaxy.yml` is not still `https://github.com/rhpds/your-project-repo` (the
shipped placeholder) and does resolve to this project's actual repo.

### G-4 — role meta author should not be a placeholder

- **Severity:** LOW
- **Auto-fix:** Yes — set to the GitHub username from `catalog-info.yaml`/`spec.yaml`

Check `galaxy_info.author` in each `automation/ansible/roles/*/meta/main.yml` is not still
`<Your Name>`.

---

## ZT-Specific Rules (Z-rules) — Skip if pattern is NOT ZT Guided

### Z-1 — lab-metadata.yml must exist at repo root

- **Severity:** HIGH
- **Auto-fix:** No — requires author to provide lab metadata

ZT repos use `lab-metadata.yml` for lab name, shortname, maintainer, description, and git_ref (production/development branches). This file is required for ZT catalog integration.

### Z-2 — lab-metadata.yml must have lab.name and lab.shortname

- **Severity:** MEDIUM
- **Auto-fix:** No — requires author input

Both fields must be non-empty and not placeholder values.

### Z-3 — lab-metadata.yml must have lab.git_ref.production

- **Severity:** HIGH
- **Auto-fix:** No — requires author to specify the production branch/tag

The production git reference is required for catalog deployment. Development ref defaults to `main` if not specified.

---

## Module & Page Coverage Rules (J-rules) — PH projects only

These rules mirror central-api's Group J checks (`development_checks.py`), which only run once at final
submission. Running the same checks locally via config-reviewer lets authors catch mismatches early,
during normal development, instead of at the submission gate. Skip this section entirely for non-PH
projects (no `publishing-house/spec.yaml`).

### J-02 — Every module outline must have a matching page

- **Severity:** HIGH
- **Auto-fix:** Partial — can offer to rename the page (or fix the `nav.adoc` xref) but needs author confirmation

Each file in `publishing-house/spec/modules/*.md` must have a corresponding `.adoc` file in
`content/modules/ROOT/pages/` with the **same filename stem** — not just a matching `module-NN` prefix.
Example: outline `module-01-what-is-a-quickstart.md` requires page `module-01-what-is-a-quickstart.adoc`.
This is how `writer-helper` derives filenames automatically (replace `.md` with `.adoc`); mismatches
happen only when an author writes or renames files by hand.

Only flag an outline as missing its page if the corresponding module's `status` in `spec.yaml` is
`in_progress` or `complete` — an outline for a `not_started` module with no page yet is expected, not
a finding.

### J-03 — index.adoc must exist

- **Severity:** HIGH
- **Auto-fix:** Yes — create a minimal stub

`content/modules/ROOT/pages/index.adoc` is created at scaffold time (via `scaffold.py` or config-helper
Route B) and should exist for the entire development lifecycle. Check unconditionally on every run.

### J-04 — conclusion.adoc must exist (gated on all modules complete)

- **Severity:** HIGH
- **Auto-fix:** No — redirect to `writer-helper` instead; a stub isn't useful since it needs full module context

`content/modules/ROOT/pages/conclusion.adoc` is only generated by `writer-helper` Step 6c, after **all**
modules in `spec.yaml` show `status: complete`. This check must be gated the same way:

- If any module is not yet `complete` → **SKIP** silently. Do not report a finding — the file is not
  expected to exist yet and flagging it would just create noise for the entire development period.
- If all modules are `complete` and the file is missing → **FAIL**.

### J-05 — nav.adoc must exist

- **Severity:** HIGH
- **Auto-fix:** Yes — create a stub with an index entry

`content/modules/ROOT/nav.adoc` is created at scaffold time alongside `index.adoc` and should exist for
the entire development lifecycle. Check unconditionally on every run. (This was previously only checked
informally via the Step 1 file table — this formalizes it as a rated, auto-fixable rule.)

---

## Review Report Format

The config-reviewer produces a report organized by severity:

```
## Showroom Config Review

**Pattern detected:** Open OCP / Guided VM / ZT Guided / Unknown
**Theme:** rhdp_showroom_theme / nookbag-bundle

### CRITICAL
- [X-4] Theme/mode mismatch: site.yml uses nookbag-bundle but ui-config.yml has `type: showroom`
- [G-2] galaxy.yml namespace `3tier_lab` starts with a digit — invalid Ansible namespace

### HIGH
- [S-2] start_page `foo::index.adoc` does not match antora.yml name `modules`
- [U-4] Tab "Terminal" has both `url` and `path` — remove one
- [J-02] No matching page for outline `module-02-secure-agent-workspace-deploy-explore.md` (module status: complete)

### MEDIUM
- [U-4b] Tab "Terminal" has `path: /wetty` but no `port` — add `port: 443`
- [U-6] Tab "OCP Console" uses `{DOMAIN}` — should be `${DOMAIN}`
- [A-5] `experimental: true` not set — UI macros will not render

### LOW
- [S-6] Site title is still the default "Showroom Template Demo"
- [U-5] Placeholder tab "Placeholder" should be replaced before deployment

### PASSED
- [S-1] Bundle URL is valid (rhdp_showroom_theme v2.0.3)
- [S-3] Content source path correct
- [X-1] start_page matches antora.yml name
- [J-03] index.adoc exists
- [J-05] nav.adoc exists
...
```

Each finding includes:

- Rule ID and description
- Current value (what was found)
- Expected value (what it should be)
- Auto-fix available flag
- Fix suggestion
