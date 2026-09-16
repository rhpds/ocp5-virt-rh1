# Finalize and Submit

Phase 6 of the intake flow. Present the final state, flag concerns, generate supporting
files, and submit to Central API.

## Step 1: Final Review

Do NOT re-ask questions that were already answered in earlier phases. Prerequisites,
RCARS vetting, and assessment strategy were all captured during Phases 2-5.

Read `publishing-house/spec.yaml` and `publishing-house/spec/design.md`. Check for any
fields that are still empty or still have placeholder values. If everything is populated,
present the summary. If anything is missing, call it out specifically.

Present the final state to the author:

> "Here's what we have before submitting for review:
>
> **Design:** `publishing-house/spec/design.md` — [title], [module count] modules, [duration]
> **Infrastructure:** [cloud provider, topology, key sizing details]
>
> [If any fields are empty or still placeholders, list them here:]
> **Still needed:**
> - [field name] — [what's missing]
>
> Review the design doc and let me know if anything needs changing before I submit."

**Wait for explicit confirmation.** Do NOT auto-proceed.

- **If feedback** → address it, update files, re-present
- **If the author flags missing fields** → ask about those specific gaps only
- **If approved** → immediately execute Steps 2-6 WITHOUT asking again

## Step 2: Generate mkdocs.yml

Generate `mkdocs.yml` at the repo root for RHDH TechDocs rendering.

Run silently:
```bash
python3 -c "
import yaml, glob, os
from pathlib import Path

spec = yaml.safe_load(Path('publishing-house/spec.yaml').read_text()) or {}
title = spec.get('spec', {}).get('title', spec.get('project', {}).get('slug', 'Publishing House Project'))

modules = sorted(glob.glob('publishing-house/spec/modules/module-*.md'))
index_lines = [f'# {title}', '', 'Welcome to the project spec. Use the navigation to browse the design and module outlines.', '', '- [Design Spec](design.md)']
for m in modules:
    fname = os.path.basename(m)
    parts = fname.replace('.md', '').split('-', 2)
    num = int(parts[1]) if len(parts) > 1 else 0
    label = parts[2].replace('-', ' ').title() if len(parts) > 2 else fname
    index_lines.append(f'- [Module {num} - {label}](modules/{fname})')
Path('publishing-house/spec/index.md').write_text(chr(10).join(index_lines) + chr(10))

nav = [{'Home': 'index.md'}, {'Design Spec': 'design.md'}]
if modules:
    mod_nav = []
    for m in modules:
        fname = os.path.basename(m)
        parts = fname.replace('.md', '').split('-', 2)
        num = int(parts[1]) if len(parts) > 1 else 0
        label = parts[2].replace('-', ' ').title() if len(parts) > 2 else fname
        mod_nav.append({f'Module {num} - {label}': f'modules/{fname}'})
    nav.append({'Modules': mod_nav})

mkdocs = {
    'site_name': title,
    'docs_dir': 'publishing-house/spec',
    'nav': nav,
    'plugins': ['techdocs-core'],
}
with open('mkdocs.yml', 'w') as f:
    yaml.dump(mkdocs, f, default_flow_style=False, sort_keys=False)
print('mkdocs.yml created')
"
```

## Step 3: Author Checkpoint

> "Your spec is ready for submission. Take a moment to review
> `publishing-house/spec/design.md` and the module outlines in
> `publishing-house/spec/modules/`.
>
> **Are you happy with this and ready to submit for review?**"

**Wait for the author's response. Do NOT auto-proceed.**

- **If feedback** → update the spec, re-validate, ask again
- **If approved** → immediately execute Steps 4-6 WITHOUT asking again

## Step 4: Commit and push

```bash
git add publishing-house/ mkdocs.yml catalog-info.yaml
git commit -m "feat: intake complete — design spec and module outlines"
git push
```

**Run this immediately. Do NOT ask the author.**

## Step 5: Submit intake to Central API

**If offline → defer:**
> "Your spec is complete and committed. When the platform is available, run
> `/rhdp-publishing-house` again to submit through the review gate."
>
> **STOP.**

**If online:**

```bash
python publishing-house/tools/ph-intake.py 2>&1
```

**Run this immediately. Do NOT ask the author.**

The response shape:
```json
{"status": <int>, "stage": "<stage or null>", "error": "<msg or null>", "validation": <object or null>}
```

Parse by `status`:

- **201** — Success. Show the review status message from the orchestrator (three pending stages: Content Review, Infra Review, Staging). Then: "Run `/rhdp-publishing-house` to check the current stage."
- **422** — Validation failed. `validation.results` contains the failed checks.
  - Show each failed check to the author: check ID, message, and field path
  - Propose specific fixes for each issue and **ask the author to confirm before making any changes**
  - Do NOT edit files, commit, or push until the author approves the proposed fixes
  - After the author approves and fixes are applied, commit, push, and re-run `ph-intake.py`
  - Loop until validation passes
- **409** — Workflow is not in intake stage. Show `error` and **STOP.**
- **404** — No workflow found. Show `error` and **STOP.**
- **Any other status** — Show the `error` message and **STOP.**

## Step 5b: Project structure cleanup

Check `project.showroom_type` in spec.yaml:

- **If `classic`** (or empty/unset): Remove Zero-Touch directories silently:
  ```bash
  rm -rf runtime-automation/ setup-automation/
  git add -A runtime-automation/ setup-automation/ 2>/dev/null || true
  git commit -m "chore: remove zero-touch dirs (classic showroom)" 2>/dev/null || true
  git push 2>/dev/null || true
  ```
- **If `zero_touch`**: Keep `runtime-automation/` and `setup-automation/` in place.

## Step 6: Report result

- If submission succeeded → show the review status message from the orchestrator (three pending stages: Content Review, Infra Review, Staging) and "Run `/rhdp-publishing-house` to check the current stage."
- If submission failed → show the error from the response.

**Return to the orchestrator** (if dispatched) or **STOP** (if invoked directly).
