# Showroom Lab Patterns Reference

## Overview

Two content modes set via `type` in `ui-config.yml`:

- `type: showroom` — free-form browse, rhdp_showroom_theme, TOC + next/prev in content pane
- `type: zerotouch` — guided linear, nookbag-bundle, Nookbag handles navigation externally (progress bar, Next/Previous, Solve, Skip, Exit)

Theme MUST match content mode. Mismatch produces duplicate or missing navigation.

---

## Pattern 1: Open (classic)

- `type: showroom`
- Theme: `rhdp_showroom_theme`
- site.yml bundle: `https://github.com/rhpds/rhdp_showroom_theme/releases/download/v2.0.3/ui-bundle.zip` (or `latest`)
- ui-config.yml uses `type: showroom` header with `default_width`, `persist_url_state`, `view_switcher`
- No `antora:` module labels block
- No `runtime-automation/` directory
- Best for: self-paced workshops, demos, reference environments

### Infrastructure variants

**Open OCP**: tabs include OCP Console (`url: 'https://console-openshift-console.${DOMAIN}'`), terminal (`path: /wetty` + `port: 443`)

**Open VM**: tabs include stacked terminals (primary Bastion `path: /wetty` + `port: 443`, secondary Worker `secondary_path: /terminal2` + `secondary_port: 443`)

### Complete Open OCP ui-config.yml

```yaml
---
type: showroom

default_width: 30
persist_url_state: true

view_switcher:
  enabled: true
  default_mode: split

tabs:
  - name: OCP Console
    url: 'https://console-openshift-console.${DOMAIN}'
  - name: Bastion ${USER}
    path: /wetty
    port: 443
  - name: Placeholder
    url: /placeholder
```

### Complete Open VM ui-config.yml

```yaml
---
type: showroom

default_width: 30
persist_url_state: true

view_switcher:
  enabled: true
  default_mode: split

tabs:
  - name: Bastion
    path: /wetty
    port: 443
    secondary_name: Worker
    secondary_path: /terminal2
    secondary_port: 443
  - name: Placeholder
    url: /placeholder
```

### Open site.yml (same for both OCP and VM)

```yaml
---
site:
  title: Showroom Template Demo
  url: https://github.com/rhpds/showroom-template
  start_page: modules::index.adoc
content:
  sources:
    - url: .
      start_path: content
ui:
  bundle:
    url: https://github.com/rhpds/rhdp_showroom_theme/releases/download/v2.0.3/ui-bundle.zip
    snapshot: true
antora:
  extensions:
    - require: '@sntke/antora-mermaid-extension'
      mermaid_library_url: https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs
      script_stem: header-scripts
      mermaid_initialize_options:
        start_on_load: true
    - require: '@andrew-jones/antora-tabs-extension'
output:
  dir: ./www
```

---

## Pattern 2: Guided (AgD v2)

- `type: zerotouch`
- Theme: `nookbag-bundle`
- site.yml bundle: `https://github.com/rhpds/nookbag-bundle/releases/download/v0.0.3/ui-bundle.zip`
- site.yml includes dev-mode extension: `- require: /antora/lib/dev-mode.js` with `enabled: true`
- ui-config.yml uses `antora:` block with module name/label mappings (drives solve/validate button UI)
- Requires `runtime-automation/` directory with `solve.yml` + `validate.yml` per module (optional `setup.yml` for module-01)
- Best for: instructor-led or structured labs requiring sequential completion and validation

### Infrastructure variants

**Guided OCP**: tabs include OCP Console + terminal (`path: /wetty` + `port: 443`)

**Guided VM**: tabs include stacked terminals (Bastion `path: /wetty` + `port: 443`, Worker `secondary_path: /terminal2` + `secondary_port: 443`)

### Complete Guided OCP ui-config.yml

```yaml
---
antora:
  name: modules
  dir: www
  modules:
    - name: index
      label: "Introduction"
    - name: module-01
      label: "Module 1: Your First Lab Page"
    - name: module-02
      label: "Module 2: Another Module"

tabs:
  - name: OCP Console
    url: 'https://console-openshift-console.${DOMAIN}'
  - name: ">_ terminal"
    path: /wetty
    port: 443
  - name: Placeholder
    url: /placeholder
```

### Complete Guided VM ui-config.yml

```yaml
---
antora:
  name: modules
  dir: www
  modules:
    - name: index
      label: "Introduction"
    - name: module-01
      label: "Module 1: Your First Lab Page"
    - name: module-02
      label: "Module 2: Another Module"

tabs:
  - name: ">_ Bastion"
    path: /wetty
    port: 443
    secondary_name: Worker
    secondary_path: /terminal2
    secondary_port: 443
  - name: Placeholder
    url: /placeholder
```

### Guided site.yml (same for both OCP and VM)

```yaml
---
site:
  title: Showroom Template Demo
  url: https://github.com/rhpds/showroom-template
  start_page: modules::index.adoc
content:
  sources:
    - url: .
      start_path: content
ui:
  bundle:
    url: https://github.com/rhpds/nookbag-bundle/releases/download/v0.0.3/ui-bundle.zip
    snapshot: true
antora:
  extensions:
    - require: '@sntke/antora-mermaid-extension'
      mermaid_library_url: https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs
      script_stem: header-scripts
      mermaid_initialize_options:
        start_on_load: true
    - require: '@andrew-jones/antora-tabs-extension'
    - require: /antora/lib/dev-mode.js
      enabled: true
output:
  dir: ./www
```

### Guided runtime-automation structure

```
runtime-automation/
├── module-01/
│   ├── setup.yml      # optional, runs once before the module
│   ├── solve.yml      # runs when user clicks Solve
│   └── validate.yml   # runs when user clicks Validate
└── module-02/
    ├── solve.yml
    └── validate.yml
```

The `antora.modules` entries in ui-config.yml must match the page filenames in `content/modules/ROOT/pages/`. The `name` field is the filename without `.adoc`, and the `label` is the display name in the Nookbag UI.

---

## Pattern 3: ZT Guided (Zero Touch)

- Same as Guided but with additional infrastructure config
- `type: zerotouch`, nookbag-bundle theme
- site.yml is otherwise the same as Guided, **except** it does NOT include the
  `@sntke/antora-mermaid-extension` / `@andrew-jones/antora-tabs-extension` entries under
  `antora.extensions` — Project Zero infra doesn't support these extensions yet. The author can
  add them back manually once it does; do not add them during scaffolding.
- ui-config.yml same format as Guided, including the terminal tab — `path: /wetty` + `port: 443`, same as every other pattern
- Additional directories beyond Guided:
  - `config/instances.yaml` — VM definitions (image, memory, cores, tags, networks)
  - `config/networks.yaml` — network definitions
  - `config/firewall.yaml` — egress rules
  - `setup-automation/setup.yml` — environment setup playbook (runs once on pod creation)
- Best for: labs running on Project Zero infrastructure

### Complete ZT site.yml

```yaml
---
site:
  title: Showroom Template Demo
  url: https://github.com/rhpds/showroom-template
  start_page: modules::index.adoc
content:
  sources:
    - url: .
      start_path: content
ui:
  bundle:
    url: https://github.com/rhpds/nookbag-bundle/releases/download/v0.0.3/ui-bundle.zip
    snapshot: true
output:
  dir: ./www
```

Note the absence of an `antora.extensions` block — see above.

### Complete ZT ui-config.yml

```yaml
---
antora:
  name: modules
  dir: www
  modules:
    - name: index
      label: "Introduction"
    - name: module-01
      label: "Module 1: Your First Lab Page"
    - name: module-02
      label: "Module 2: Another Module"

tabs:
  - name: ">_ terminal"
    path: /wetty
    port: 443
  - name: Placeholder
    url: /placeholder
```

### ZT config directory examples

```yaml
# config/instances.yaml
---
containers: []
virtualmachines:
  - name: "host1"
    image: "rhel-9.5"
    memory: "2G"
    cores: 1
    image_size: "40G"
    tags:
      - key: "AnsibleGroup"
        value: "bastions"
    networks:
      - default
      - secondary
```

```yaml
# config/networks.yaml
---
- name: default
- name: secondary
```

```yaml
# config/firewall.yaml
---
egress:
  - ports:
      - protocol: TCP
        port: 443
```

---

## Pattern detection rules

How to detect which pattern is in use from existing config:

1. If ui-config.yml has `type: showroom` → Open pattern
2. If ui-config.yml has `antora:` block (with or without `type: zerotouch`) → Zerotouch pattern (Guided or ZT)
3. If zerotouch AND `config/` directory exists → ZT Guided
4. If zerotouch AND no `config/` directory → Guided (AgD v2)
5. Bundle URL in site.yml confirms: `rhdp_showroom_theme` = Open, `nookbag-bundle` = Zerotouch

---

## Publishing House integration

When a PH project has `.scaffolds/` directory and `scaffold.py`:

- The intake writes `project.showroom_type` to `publishing-house/spec.yaml` during Phase 1 (discovery)
- config-helper reads this field and maps it to the scaffold pattern:

| `project.showroom_type` | scaffold.py `--pattern` | Content mode | Theme |
|--------------------------|-------------------------|--------------|-------|
| `classic` | `agd-open` | `type: showroom` | rhdp_showroom_theme |
| `zero_touch` | `zt-guided` | `type: zerotouch` | nookbag-bundle |
| `guided` | `agd-guided` | `type: zerotouch` | nookbag-bundle |

Note: `agd-guided` is not currently offered during intake. Only `classic` and `zero_touch` are selectable.

- config-helper confirms the detected pattern with the user, then runs `scaffold.py --pattern <name> --force`
- scaffold.py copies pattern-specific stubs, sets `showroom_type` and `infrastructure` in spec.yaml, deletes `.scaffolds/`
- After scaffold.py runs, config-helper operates in modification mode on the generated files

### Migrated repos (`project.intake_type: migration`)

A migrated repo already has real `content/`, `runtime-automation/`, `setup-automation/`,
`config/`, and `ui-config.yml` — imported from the source repo before intake ran. config-helper
passes `--migration` to `scaffold.py` for these projects (see `config-helper.md` Route A, steps
A.1b and A.2):

- `scaffold.py --migration` never overwrites a file that already exists. `runtime-automation/`,
  `setup-automation/`, and `config/` are fully hands-off — never wiped/recreated, and never even
  scanned for fill-in — since a migrated repo structures them completely differently from the
  fresh scaffold (shell scripts in real `module-NN-<slug>/` folders, not the
  ansible-playbook-per-module placeholders `zt-guided` ships). Other pattern files (`site.yml`,
  `ui-config.yml`) still get filled in from `common/` and the pattern dir if genuinely missing.
- If `.scaffolds/<pattern>-migration/` exists (today only `zt-guided-migration/`), its contents
  are overlaid on top afterward, always overwriting — this is what swaps in the migration-aware
  `qa-automation/` (drives the legacy `runtime-automation/<module>/{solve,validation}-<host>.sh`
  shell scripts a migrated repo already ships, instead of the ansible-playbook-per-module style
  `qa-automation/` that fresh scaffolds get).
- The `migrate` skill's Phase 2 already renames `content/` pages, `runtime-automation/` dirs,
  and `nav.adoc` xrefs to match the `module-NN-<slug>` outlines *during intake*, as soon as
  they're generated (see `migrate/procedures/02-module-outlines-from-content.md`) — but it
  never touches `ui-config.yml`. Before scaffold.py runs, **A.1b** aligns `ui-config.yml`'s
  `antora.modules` names to match (the one thing intake doesn't do), falling back to renaming
  files/folders itself only for anything intake's Phase 2 missed — positionally, using
  `ui-config.yml`'s existing `antora.modules` order (excluding `index`) zipped against the sorted
  outline filenames.

---

## Common antora.yml (shared across all patterns)

```yaml
name: modules
title: My Lab Title
version: ~
nav:
  - modules/ROOT/nav.adoc

asciidoc:
  attributes:
    experimental: true
    page-pagination: true
    guid: '%GUID%'
    ssh_user: lab-user
    ssh_password: changeme
    openshift_console_url: 'https://console-openshift-console.apps.cluster-%GUID%.example.com'
    openshift_cluster_ingress_domain: 'apps.cluster-%GUID%.example.com'
```

---

## Minimum directory layout for all patterns

```
your-content-repo/
├── content/
│   ├── antora.yml
│   └── modules/ROOT/
│       ├── nav.adoc
│       └── pages/
│           ├── index.adoc
│           └── module-01.adoc
├── site.yml
└── ui-config.yml
```
