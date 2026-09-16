## Ansible Collection Conventions for RHDP Labs

Conventions for the starter Ansible collection scaffolded into `automation/ansible/` (via
`scaffold.py --automation ansible` or `--automation both`). Used by `config-helper` to fill in
real project values after scaffolding, and by `config-reviewer` to validate the result
(G-rules in @rhdp-publishing-house/skills/development/references/validation-rules.md).

## Data Sources

Read in this order — use the first one that has the value:

| Value | Primary source | Fallback |
|---|---|---|
| Repo name | `catalog-info.yaml` `metadata.name` | `publishing-house/spec.yaml` `project.slug` |
| GitHub username | `catalog-info.yaml` annotation `ph.rhdp.io/github-user` | none — omit from author line if missing |
| Owner email | `catalog-info.yaml` annotation `ph.rhdp.io/owner` | `publishing-house/spec.yaml` `project.owner_email` |

`catalog-info.yaml` only exists for projects created through the RHDH scaffolder — it's absent if
someone cloned `rhdp-publishing-house-template` directly without going through intake. If neither
source has a value, ask the author directly rather than writing a placeholder into a real file.

There is no real display name anywhere in the PH data model — only a GitHub username and an email.
The author line is always `"<github-user> <owner-email>"`, e.g. `"andrew-jones
<andrew.jones@redhat.com>"`.

The repository URL is always `https://github.com/rhpds/<repo-name>` — the GitHub org is hardcoded
`rhpds` throughout the template pipeline, never derived.

## Namespace Derivation

Ansible Galaxy requires both `namespace` and `name` in `galaxy.yml` to:

- Contain only lowercase alphanumeric characters and underscores
- Be at least 3 characters long
- Not start with an underscore or a digit
- Not contain consecutive underscores

(Source: [Ansible collections_galaxy_meta docs](https://docs.ansible.com/projects/ansible/latest/dev_guide/collections_galaxy_meta.html).
A collection or role that violates these rules will fail `ansible-galaxy collection install` for
anyone who tries to consume it.)

**Derivation:** take the repo name and replace every `-` with `_`. RHDH already constrains
`project_name` to `^[a-z0-9-]+$` at creation time (`templates/publishing-house-project/template.yaml`
in `rhdp-publishing-house`), so the input is always lowercase letters, digits, and hyphens — there
is no uppercase or special-character case to handle. That leaves exactly three ways the derived
namespace can still fail Ansible's rules:

| Repo name | Derived | Problem |
|---|---|---|
| `3tier-lab` | `3tier_lab` | starts with a digit |
| `ai` | `ai` | fewer than 3 characters |
| `foo--bar` | `foo__bar` | consecutive underscores |

If the derived namespace fails any of these checks, do **not** write it into `galaxy.yml` anyway —
ask the author for a valid namespace instead:

> The repo name `<repo-name>` doesn't map to a valid Ansible namespace (`<reason>`). What namespace
> would you like to use? Lowercase letters/digits/underscores only, 3+ characters, can't start with
> a digit or underscore, no double underscores.

## Collection Name

`name` in `galaxy.yml` is always `"automation"` — fixed, never derived, and it ships pre-filled in
the template source (`.scaffolds/automation/ansible/galaxy.yml`), so there's nothing to fill in at
scaffold time. It's not the repo name (that's what `namespace` captures); it just needs to be *a*
valid, stable identifier for "this project's automation collection." Every RHDP lab collection is
`<namespace>.automation`, where `<namespace>` is unique per project, so there's no cross-project
collision risk.

## Files and Fields Filled In

Only fields that actually vary per project need filling in at scaffold time. `name: "automation"`
and the README titles already ship correct in the template — config-helper never touches them.

| File | Field | Value |
|---|---|---|
| `automation/ansible/galaxy.yml` | `namespace` | derived (see above) |
| `automation/ansible/galaxy.yml` | `authors` | `["<github-user> <owner-email>"]` |
| `automation/ansible/galaxy.yml` | `repository` | `https://github.com/rhpds/<repo-name>` |
| `automation/ansible/roles/example/meta/main.yml` | `galaxy_info.author` | `<github-user>` |
| `automation/ansible/README.md` | FQCN example | `<namespace>.automation.*` |
| `automation/ansible/roles/example/README.md` | FQCN example | `<namespace>.automation.example` |

**Not filled in:** `license` in both `galaxy.yml` and `roles/example/meta/main.yml`. There's no
signal anywhere in the PH data model for which license an org wants to apply, so this is left as an
explicit reminder to the author rather than guessed at.

## Adding Roles Later

Once the collection is named, any new role added under `roles/` is referenced by its fully
qualified name: `<namespace>.automation.<role_name>`. This only changes if the author renames the
project's repo after the collection was already filled in — `galaxy.yml` is not re-derived
automatically after the first fill-in; re-run the fill-in step manually if that happens.
