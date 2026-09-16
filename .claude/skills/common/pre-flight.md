# Pre-flight — Steps 1–3

**ALWAYS complete these steps before any skill-specific work.**

## Step 0 — Detect python

Run silently:
```bash
command -v python >/dev/null 2>&1 && echo "PYTHON:python" || command -v python3 >/dev/null 2>&1 && echo "PYTHON:python3" || echo "PYTHON:none"
```

Extract the python command from the `PYTHON:` line. If `none` → show: "No python found. Install Python 3 and try again." **STOP.**

Use this command (`python` or `python3`) for **all** python calls in this skill session. The remaining steps in pre-flight use `python` in code blocks — substitute with whichever was detected.

## Step 1 — Verify this is a Publishing House project

Run silently:
```bash
python -c "
from pathlib import Path
ci = Path('catalog-info.yaml')
spec = Path('publishing-house/spec.yaml')
if ci.exists() and spec.exists():
    print('ok')
elif not ci.exists():
    print('no-catalog')
else:
    print('no-spec')
"
```

- `ok` → proceed.
- `no-catalog` → show:
  > This doesn't look like a Publishing House project — `catalog-info.yaml` is missing.
  >
  > Projects must be created through the **RHDH Developer Hub** template. Open RHDH, choose the **Publishing House Content Project** template, and fill in the form. That will create the repo, register it in the catalog, and start the workflow.
  >
  > Then open the created repo in DevSpaces and run the skill again.

  **STOP.**
- `no-spec` → show: "`publishing-house/spec.yaml` is missing. This repo may not have been scaffolded correctly." **STOP.**

## Step 2 — Read project identity

Run silently:
```bash
python -c "
import yaml
from pathlib import Path
spec = yaml.safe_load(Path('publishing-house/spec.yaml').read_text()) or {}
pid = spec.get('project', {}).get('slug', '')
print(f'project_id:{pid}')
"
```

Extract `project_id`. If empty → show error: "`project.slug` is missing in `spec.yaml`." **STOP.**

## Step 3 — Check auth

Run silently:
```bash
python -c "
import json, os, yaml
f = os.path.expanduser('~/.config/publishing-house/auth.json')
central = ''
try:
    ci = yaml.safe_load(open('catalog-info.yaml'))
    for link in ci.get('metadata', {}).get('links', []):
        if link.get('title') == 'Central':
            central = link['url']
            break
except Exception:
    pass
if not central:
    print('no-central')
else:
    os.makedirs(os.path.dirname(f), exist_ok=True)
    d = json.load(open(f)) if os.path.exists(f) else {}
    d['central'] = central
    with open(f, 'w') as fh:
        json.dump(d, fh, indent=2)
    os.chmod(f, 0o600)
    cred = d.get('credential', '')
    print(f'cred:{cred[:8]}' if cred else 'no-cred')
    print(f'central:{central}')
"
```

Extract `central_url` from the `central:` line.

- Has `cred:` and `central:` → proceed.
- `no-central` → show: "Cannot find Central API URL. Check that `catalog-info.yaml` has a **Central** link." **STOP.**
- `no-cred` → show:

  > **You need a Publishing House API key.**
  >
  > Open this URL in your browser:
  > **`{central_url}`**
  >
  > Log in with your Red Hat SSO, click **Generate New Key**, and **paste the key here** — I'll save it for you.

  Then try to open the browser (works locally, silently fails in DevSpaces):
  ```bash
  python -c "import subprocess; subprocess.Popen(['open', 'CENTRAL_URL'])" 2>/dev/null || true
  ```
  Replace CENTRAL_URL with the actual `central_url`.

  Wait for the author to paste the key. Once received, save it:
  ```bash
  python -c "
import json, os
key = 'PASTE_KEY_HERE'
path = os.path.expanduser('~/.config/publishing-house/auth.json')
d = json.load(open(path)) if os.path.exists(path) else {}
d['credential'] = key
os.makedirs(os.path.dirname(path), exist_ok=True)
with open(path, 'w') as f:
    json.dump(d, f, indent=2)
os.chmod(path, 0o600)
print('saved')
"
  ```
  Replace PASTE_KEY_HERE with the actual key. Confirm: > Got it — you're all set.
