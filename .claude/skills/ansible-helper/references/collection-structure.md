# Ansible Collection Structure Reference

## Collection layout

```
automation/ansible/
├── galaxy.yml
├── README.md
├── meta/
│   └── runtime.yml
└── roles/
    └── <role_name>/
        ├── tasks/main.yml
        ├── defaults/main.yml
        ├── meta/main.yml
        └── README.md
```

Optional directories (add only when needed):
- `<role_name>/vars/main.yml` — role-internal variables not exposed to callers
- `<role_name>/handlers/main.yml` — notification handlers

## galaxy.yml

```yaml
namespace: <namespace>
name: ansible
version: 1.0.0
readme: README.md
authors:
- <owner_email>
description: Ansible collection for <slug> lab automation
license:
- GPL-2.0-or-later
license_file: ''
tags: []
dependencies: {}
repository: ''
documentation: ''
homepage: ''
issues: ''
build_ignore: []
```

## meta/runtime.yml

```yaml
#SPDX-License-Identifier: MIT-0
---
# requires_ansible: '>=2.9.10'
```

## Role stub files

### tasks/main.yml
```yaml
#SPDX-License-Identifier: MIT-0
---
# tasks file for <role_name>
```

### defaults/main.yml
```yaml
#SPDX-License-Identifier: MIT-0
---
# defaults file for <role_name>
```

### meta/main.yml
```yaml
#SPDX-License-Identifier: MIT-0
galaxy_info:
  author: <owner_email>
  description: <role_description>
  license: GPL-2.0-or-later
  min_ansible_version: 2.9
  galaxy_tags: []
dependencies: []
```

### README.md
```markdown
# <role_name>

<role_description>

## Requirements

None.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| | | |

## Dependencies

None.

## Example Playbook

\`\`\`yaml
- hosts: all
  roles:
    - role: <namespace>.ansible.<role_name>
\`\`\`

## License

GPL-2.0-or-later
```
