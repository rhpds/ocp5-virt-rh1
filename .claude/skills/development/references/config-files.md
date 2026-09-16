# Showroom Config Files Reference

## site.yml (Antora Playbook)

The Antora playbook file. Content developers should NOT modify this file after initial setup — content-specific config belongs in `content/antora.yml`.

### Structure

```yaml
---
site:
  title: <Lab Title>
  url: <repo URL>
  start_page: modules::index.adoc

content:
  sources:
    - url: .
      start_path: content

ui:
  bundle:
    url: <theme bundle URL>
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

### Field reference

| Field | Description |
|-------|-------------|
| `site.title` | Display title for the built site |
| `site.url` | Repository URL |
| `site.start_page` | Entry page. Must match the `name` field in `content/antora.yml` (e.g. `modules::index.adoc`) |
| `content.sources[].url` | `.` for local builds |
| `content.sources[].start_path` | Path to the directory containing `antora.yml`. Always `content` |
| `ui.bundle.url` | Theme bundle URL. Must match the content mode |
| `ui.bundle.snapshot` | `true` to re-fetch the bundle on `--fetch` |
| `antora.extensions` | Registered Antora extensions — see "Mermaid + tabs extensions" and "Dev-mode extension" below; **not every pattern gets the same set** |
| `output.dir` | Build output directory. Must match `antora.dir` in `ui-config.yml` for zerotouch patterns |

### Theme bundle options

**rhdp_showroom_theme** (for `type: showroom` / Open pattern):

- Tagged: `https://github.com/rhpds/rhdp_showroom_theme/releases/download/v2.0.3/ui-bundle.zip`
- Rolling: `https://github.com/rhpds/rhdp_showroom_theme/releases/download/latest/ui-bundle.zip`

**nookbag-bundle** (for `type: zerotouch` / Guided and ZT Guided patterns):

- Tagged: `https://github.com/rhpds/nookbag-bundle/releases/download/v0.0.3/ui-bundle.zip`

Theme MUST match content mode. Using `rhdp_showroom_theme` with a zerotouch ui-config or vice versa produces duplicate or missing navigation.

### Navbar branding (rhdp_showroom_theme only)

```yaml
site:
  keys:
    navbar_logo: summit
```

Options for `navbar_logo`: `summit`, `rhdp`, or omit entirely for default RHDP branding.

### Mermaid + tabs extensions (Open and Guided patterns — NOT ZT Guided)

`Open` (agd-open) and `Guided` (agd-guided) site.yml files register the mermaid and tabs
extensions:

```yaml
antora:
  extensions:
    - require: '@sntke/antora-mermaid-extension'
      mermaid_library_url: https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs
      script_stem: header-scripts
      mermaid_initialize_options:
        start_on_load: true
    - require: '@andrew-jones/antora-tabs-extension'
```

**ZT Guided (`zt-guided`) does NOT get this block during scaffolding** — Project Zero infra
doesn't support these extensions yet. The author can add it back manually to `site.yml` once
support lands; do not add it during scaffolding in the meantime.

### Dev-mode extension (zerotouch patterns only)

Zerotouch site.yml files include the dev-mode extension:

```yaml
antora:
  extensions:
    - require: /antora/lib/dev-mode.js
      enabled: true
```

Dev-mode behavior:

- Adds an Attributes reference page listing every AsciiDoc attribute
- Shows unlisted pages under a Dev Mode section
- Injected at build time by the Showroom container — NOT available during local Antora builds
- Only register in zerotouch pattern site.yml files

---

## ui-config.yml

Controls the right-hand pane of the Showroom interface. Two formats exist depending on content mode.

### Format 1: `type: showroom` (Open pattern)

```yaml
---
type: showroom

default_width: 30
persist_url_state: true

view_switcher:
  enabled: true
  default_mode: split

tabs:
  - name: <Tab Name>
    url: <URL>
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Must be `showroom` |
| `default_width` | integer | Right pane width as a percentage. Default: `30` |
| `persist_url_state` | boolean | Remember the active tab on page refresh |
| `view_switcher.enabled` | boolean | Allow content-only / split / tabs-only toggle |
| `view_switcher.default_mode` | string | `split`, `content`, or `tabs` |
| `tabs` | list | Tab definitions (see tab properties below) |

### Format 2: `antora:` block (Zerotouch / Guided patterns)

```yaml
---
antora:
  name: modules
  dir: www
  modules:
    - name: index
      label: "Introduction"
    - name: module-01
      label: "Module 1: Title"

tabs:
  - name: <Tab Name>
    url: <URL>
```

| Field | Type | Description |
|-------|------|-------------|
| `antora.name` | string | Must match `name` in `content/antora.yml` |
| `antora.dir` | string | Must match `output.dir` in `site.yml` (without leading `./`) |
| `antora.modules` | list | Module labels for Nookbag navigation |
| `antora.modules[].name` | string | Page filename without `.adoc`. Must match a file in `content/modules/ROOT/pages/` |
| `antora.modules[].label` | string | Display name in the Nookbag progress bar and navigation UI |
| `tabs` | list | Tab definitions (see tab properties below) |

The `antora.modules` entries drive the Nookbag progress bar and solve/validate buttons. Each `name` must correspond to both a page file and (for guided patterns) a `runtime-automation/<name>/` directory.

### Tab properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Tab display name (required) |
| `url` | string | Full URL or relative path for the tab iframe. Supports variable substitution |
| `path` | string | Path relative to the Showroom instance (e.g. `/wetty`). Use for services in the same pod/host |
| `port` | integer | Port for constructing tab URL from `path`. Set explicitly — typically `443` for standard HTTPS — even though 443 is technically the default; older/production Showroom UI builds don't reliably support omitting it |
| `secondary_name` | string | Display name for stacked secondary panel within the same tab |
| `secondary_path` | string | Path for the secondary panel |
| `secondary_port` | integer | Port for the secondary panel. Same rule as `port` — set explicitly (typically `443`) |

Rules:

- Each tab needs either `url` OR `path` (not both, not neither)
- Every `path`/`secondary_path` tab should set an explicit `port`/`secondary_port` — typically `443` — for backward compatibility. Only `url` tabs can omit it (the URL is already fully specified)
- `secondary_*` properties create a stacked split within one tab (top/bottom)

### Variable substitution in tab URLs

Tab URLs support `${VARIABLE}` syntax. Variables are substituted at deploy time via `envsubst` in the content container entrypoint.

**Built-in variables** (always available):

| Variable | Source | Description |
|----------|--------|-------------|
| `${DOMAIN}` | Helm `deployer.domain` | Apps domain of the OpenShift cluster (e.g. `apps.cluster.example.com`) |
| `${GUID}` | Helm `guid` | Unique environment identifier |
| `${USER}` | Helm `user` | Current user identifier (multi-user deployments) |

**Custom variables** — authors can define additional variables in `content/antora.yml` under `asciidoc.attributes.environment_variables`. These are exported as environment variables before `envsubst` runs:

```yaml
asciidoc:
  attributes:
    environment_variables:
      CLUSTER_NAME: "my-cluster"
      APP_NAMESPACE: "my-app"
```

These become available as `${CLUSTER_NAME}` and `${APP_NAMESPACE}` in `ui-config.yml`.

IMPORTANT: Use `${DOMAIN}` not `{DOMAIN}`. The `${}` syntax is required. Antora attribute substitution in AsciiDoc uses `{attribute}` without `$`, but ui-config.yml uses `${VARIABLE}`.

IMPORTANT: Avoid using common shell variables (e.g. `${HOME}`, `${PATH}`, `${GIT_REPO_URL}`) in tab URLs — `envsubst` replaces ALL environment variables in the container, not just the intended ones. Stick to `DOMAIN`, `GUID`, `USER`, and explicitly defined custom variables.

### Common tab patterns

**Terminal (Wetty — SSH to bastion):**

```yaml
- name: ">_ terminal"
  path: /wetty
  port: 443
```

Same syntax for every pattern, including ZT Guided — there's no infra-specific variant anymore.

**OCP Terminal (ttyd with oc CLI, no bastion required):**

```yaml
- name: ">_ OCP Terminal"
  url: 'https://codeserver-codeserver.${DOMAIN}'
```

**Two terminals (stacked vertically in one tab):**

```yaml
- name: ">_ Terminals"
  path: /wetty
  port: 443
  secondary_name: Worker
  secondary_path: /terminal2
  secondary_port: 443
```

Vertical split is commonly used to run a `watch` command in one terminal while working in the other, or to SSH to different hosts simultaneously.

**OpenShift Console:**

```yaml
- name: OCP Console
  url: 'https://console-openshift-console.${DOMAIN}'
```

**OCP application services (common routes):**

```yaml
- name: ArgoCD
  url: 'https://openshift-gitops-server-openshift-gitops.${DOMAIN}'
- name: RHACS
  url: 'https://central-stackrox.${DOMAIN}'
- name: Developer Hub
  url: 'https://backstage-developer-hub-backstage.${DOMAIN}'
- name: Grafana
  url: 'https://grafana-grafana.${DOMAIN}'
```

Route URL pattern: `https://<route-name>-<namespace>.${DOMAIN}`. If the author doesn't know the route yet, use a placeholder and fill it in during development.

**External URL:**

```yaml
- name: Product Docs
  url: 'https://docs.redhat.com'
```

External sites with `X-Frame-Options` or `Content-Security-Policy` headers blocking iframes will show a blank pane.

**Placeholder (local development):**

```yaml
- name: Placeholder
  url: /placeholder
```

Renders a stub panel for local preview without running real services.

---

## content/antora.yml (Component Descriptor)

Defines the Antora component metadata and AsciiDoc attributes. This is where content-specific configuration belongs.

### Structure

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

### Field reference

| Field | Description |
|-------|-------------|
| `name` | Component name. Must match `site.yml` `start_page` prefix and `ui-config.yml` `antora.name` |
| `title` | Lab display title |
| `version` | Use `~` for versionless (standard for all Showroom content) |
| `nav` | Path to the sidebar navigation file, relative to `antora.yml` |

### Common attributes by infrastructure type

**All labs:**

| Attribute | Description |
|-----------|-------------|
| `guid` | Unique environment identifier. Always present. Default placeholder: `'%GUID%'` |
| `experimental` | Set to `true` to enable `btn:[]`, `kbd:[]`, `menu:[]` UI macros |
| `page-pagination` | Set to `true` to enable next/prev page links |
| `lab_name` | Optional. Used in page titles |

**OCP labs:**

| Attribute | Description |
|-----------|-------------|
| `openshift_console_url` | OCP web console URL |
| `openshift_cluster_ingress_domain` | Apps domain for routes |

**VM labs:**

| Attribute | Description |
|-----------|-------------|
| `bastion_public_hostname` | Bastion SSH host |
| `ssh_user` | SSH username |
| `ssh_password` | SSH password |

Attribute values containing `%GUID%` or similar placeholders are replaced by `user_data` at deploy time. The defaults in `antora.yml` are for local development only.

### Page-links dropdown (rhdp_showroom_theme only)

```yaml
asciidoc:
  attributes:
    page-links:
      - url: https://docs.redhat.com
        text: Red Hat Documentation
      - url: https://access.redhat.com
        text: Red Hat Customer Portal
```

Renders a dropdown menu in the navbar with links to external resources.

### User display (rhdp_showroom_theme only)

```yaml
asciidoc:
  attributes:
    user: 'User X'
```

Shows the user name in the navbar. Typically set via `user_data` at deploy time rather than hardcoded.

---

## nav.adoc (Navigation)

Controls the sidebar table of contents. Located at `content/modules/ROOT/nav.adoc`.

### Structure

```asciidoc
* xref:index.adoc[Lab Overview]
* xref:module-01.adoc[1. Getting Started]
* xref:module-02.adoc[2. Deploy the Application]
```

### Rules

- `index.adoc` should be the first entry
- All pages in `pages/` should be listed — unlisted pages are accessible by URL but won't appear in the sidebar (unless dev-mode is enabled)
- Each xref target must exist as a file in `content/modules/ROOT/pages/`
- For zerotouch patterns, the page filenames referenced here must match the `antora.modules[].name` entries in `ui-config.yml`
- Nesting is supported with indentation (`**` for level 2, `***` for level 3)

---

## podman-compose.yaml (Local Development)

Standard local dev setup using Podman Compose. Not deployed — only used for local preview.

### Structure

```yaml
version: "3.0"
services:
  antora:
    image: quay.io/rhpds/antora:v1.3.0
    environment:
      - WAIT_FOR_GIT_CLONER=false
      - ANTORA_ENABLE_DEV_MODE=true
      - ANTORA_WATCH=true
    volumes:
      - ./:/files:z
      - www:/output:z

  httpd:
    image: registry.access.redhat.com/ubi10/httpd-24:10.1
    init: true
    ports:
      - "8080:8080"
    volumes:
      - www:/var/www/html:z
    depends_on:
      - antora
    command: ["/bin/bash", "-c", "trap '' WINCH; trap 'kill -TERM $PID' TERM; run-httpd & PID=$!; wait $PID"]

volumes:
  www:
```

### Details

- Volume mounts include `:z` for SELinux relabeling
- The `antora` service watches for file changes and rebuilds automatically (`ANTORA_WATCH=true`)
- The `httpd` service serves the built site on port 8080
- Preview at `http://localhost:8080`
- The `WAIT_FOR_GIT_CLONER=false` flag disables the git-cloner sidecar wait (not used locally)
- This file should not require modification for typical content development

---

## .github/workflows/gh-pages.yml (GitHub Pages)

Standard GitHub Actions workflow for building and deploying the Antora site to GitHub Pages.

### Details

- Usually does not need modification
- Requires Node.js 22+
- Installs the following packages:
  - `@antora/cli@3.1.15`
  - `@antora/site-generator@3.1.15`
  - `@sntke/antora-mermaid-extension@0.0.13`
  - `@andrew-jones/antora-tabs-extension@1.0.0`
- Builds the site using `npx antora site.yml`
- Deploys the `output.dir` contents to GitHub Pages

---

## Cross-file consistency requirements

These fields must be consistent across config files:

| Constraint | Files involved |
|------------|----------------|
| `name` in antora.yml must match the prefix in `site.start_page` (e.g. `modules` and `modules::index.adoc`) | `content/antora.yml`, `site.yml` |
| `name` in antora.yml must match `antora.name` in ui-config.yml (zerotouch only) | `content/antora.yml`, `ui-config.yml` |
| `output.dir` in site.yml must match `antora.dir` in ui-config.yml (zerotouch only) | `site.yml`, `ui-config.yml` |
| Theme bundle in site.yml must match the ui-config format (`rhdp_showroom_theme` for `type: showroom`, `nookbag-bundle` for zerotouch) | `site.yml`, `ui-config.yml` |
| Every `antora.modules[].name` in ui-config.yml must have a corresponding `.adoc` file in `pages/` | `ui-config.yml`, `content/modules/ROOT/pages/` |
| Every xref target in nav.adoc must exist as a file in `pages/` | `nav.adoc`, `content/modules/ROOT/pages/` |
| For zerotouch patterns, page filenames in nav.adoc must match `antora.modules[].name` entries in ui-config.yml | `nav.adoc`, `ui-config.yml` |
