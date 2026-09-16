# Development Workflow Diagram

This diagram is the authoritative reference for the RHDP Publishing House development phase.
The development skill provides a numbered dashboard for 4 workstreams (modules, automation,
e2e, health check) plus showroom config. Writing, reviewing, and automation are optional
standalone helper skills the author invokes directly.

## Mermaid Diagram

```mermaid
flowchart TD
    User([User]) --> PH["/rhdp-publishing-house\nSKILL"]

    PH --> Intake["intake phase\nrhdp-publishing-house:intake\n02-discovery to 03-design-doc\nto 04-module-outlines\nto 05-infrastructure to 06-finalize"]

    Intake --> Dev["development phase\nrhdp-publishing-house:development"]

    Dev --> S1["Step 1: Scaffold check gate"]
    S1 --> CR["Run procedures/config-reviewer.md"]
    CR -->|PASS| Dashboard
    CR -->|"FAIL, .scaffolds/ present\n(not yet scaffolded)"| AutoCH["procedures/config-helper.md\nRoute A, no ask — auto-scaffold"]
    CR -->|"FAIL, .scaffolds/ gone\n(scaffolded but broken)"| Issues["Report issues to user"]
    Issues -->|option 1: help me| CH["procedures/config-helper.md"]
    Issues -->|option 2: I'll handle it| STOP1["STOP"]
    AutoCH --> Dashboard

    Dashboard["Step 2: Development Dashboard\n1. Modules (N/M complete)\n2. GitOps Automation (if applicable)\n3. Ansible Automation (if applicable)\n4. E2E Tests\n5. Health Check\n6. Showroom Config"]

    Dashboard -->|all complete| Submit["Step 3: Submit\nph-development.py"]

    Dashboard -->|Modules| Modules["Module sub-menu\nstart/complete modules\nph-task-complete.py"]
    Dashboard -->|GitOps Automation| GitOps["1. Use GitOps helper\n2. Do it myself\n3. Back"]
    Dashboard -->|Ansible Automation| Ansible["1. Use Ansible helper\n2. Do it myself\n3. Back"]
    Dashboard -->|E2E Tests| E2E["E2E sub-menu\n1. Mark complete\n2. Back"]
    Dashboard -->|Health Check| HC["Health Check sub-menu\n1. Mark complete\n2. Back"]
    Dashboard -->|Showroom Config| Config["Showroom Config\n1. config-helper\n2. config-reviewer\n3. Back"]

    GitOps -->|no automation/gitops/ dir| CH2["procedures/config-helper.md\nAutomation Scaffolding"]
    GitOps -->|option 1| GH["[Skill] gitops-helper\nsets gitops.status: in_progress"]
    GitOps -->|option 2| GitOpsSelf["Do it myself\nsets gitops.status: in_progress"]

    Ansible -->|no automation/ansible/ dir| CH3["procedures/config-helper.md\nAutomation Scaffolding"]
    Ansible -->|option 1| AH["Ansible helper\nFUTURE - RHDPCD-110"]
    Ansible -->|option 2| AnsibleSelf["Do it myself\nsets ansible.status: in_progress"]

    Modules -.->|author invokes directly| WriterHelper["rhdp-publishing-house:writer-helper\nOPTIONAL"]
    Modules -.->|author invokes directly| ReviewerHelper["rhdp-publishing-house:reviewer-helper\nOPTIONAL"]

    WriterHelper --> MWH["[Task] rhdp-publishing-house:module-writing-helper\nAGENT"]
    ReviewerHelper --> MR["[Task] rhdp-publishing-house:module-reviewer\nAGENT"]
```

## ASCII Diagram

```
User
  |
  +- /rhdp-publishing-house (SKILL)
      |
      +- intake phase --------------------------------------------------+
      |   rhdp-publishing-house:intake                                  |
      |   +- 02-discovery -> 03-design-doc -> 04-module-outlines        |
      |      -> 05-infrastructure -> 06-finalize                        |
      +-------------------------------------------------------------------+
      |
      +- development phase ----------------------------------------------+
      |   rhdp-publishing-house:development                              |
      |                                                                  |
      |   Step 1: Scaffold check gate                                    |
      |   +- Run procedures/config-reviewer.md                          |
      |      +- PASS -> proceed to Dashboard                            |
      |      +- FAIL, .scaffolds/ present (not yet scaffolded)          |
      |      |    -> auto-scaffold via config-helper.md Route A          |
      |      |       (no ask; single scaffold-plan confirmation only)    |
      |      +- FAIL, .scaffolds/ gone (scaffolded but broken)          |
      |           -> report issues, offer numbered options               |
      |                                                                  |
      |   Step 2: Development Dashboard (numbered options)               |
      |   +- 1. Modules           -> module sub-menu (start/complete)   |
      |   +- 2. GitOps Automation -> helper or DIY (if applicable)      |
      |   +- 3. Ansible Automation -> helper or DIY (if applicable)     |
      |   +- 4. E2E Tests         -> mark complete (placeholder)        |
      |   +- 5. Health Check      -> mark complete (placeholder)        |
      |   +- 6. Showroom Config   -> config-helper / config-reviewer    |
      |                                                                  |
      |   Step 3: Submission gate                                        |
      |   +- modules + all automation children complete -> ph-development.py |
      +---------------------------------------------------------------+
      |
      +- optional helper skills (NOT dispatched by development) --------+
      |   rhdp-publishing-house:writer-helper                            |
      |   +- [Task] rhdp-publishing-house:module-writing-helper (AGENT) |
      |                                                                  |
      |   rhdp-publishing-house:reviewer-helper                          |
      |   +- [Task] rhdp-publishing-house:module-reviewer (AGENT)       |
      |                                                                  |
      |   rhdp-publishing-house:gitops-helper                             |
      |   rhdp-publishing-house:ansible-helper (FUTURE - RHDPCD-110)    |
      +---------------------------------------------------------------+
```

## Legend

| Marker | Meaning |
|---|---|
| `[Task]` | Agent invocation via Task tool with `subagent_type` parameter |
| `[Skill]` | Skill invocation via Skill tool |
| `FUTURE` | Not yet built — dispatch path is wired but destination skill does not exist yet |
| `OPTIONAL` | Not dispatched by `development` — the author invokes it directly whenever they want |

**Note:** FTL (`ftl:*`) is NOT in this diagram. Mitesh owns FTL validation separately as a standalone workflow.
