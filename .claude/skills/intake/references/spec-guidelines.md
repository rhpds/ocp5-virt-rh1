# Spec Quality Guidelines

Guidelines for evaluating and generating project specs. Used by the intake agent.

## Required Sections in design.md

A complete spec MUST have all of these (11 sections + a descriptive H1 title):

1. **H1 title** — descriptive project name (not a placeholder like `# [Project Title]`)
2. **Overview** — what the lab or demo is, why it exists, and what participants will do (direct, no flowery prose)
3. **Target Audience** — role, experience level, what they already know
4. **Prerequisites** — what the learner needs before starting; can the lab validate them?
5. **Learning Objectives** — action-verb list (Configure, Deploy, Create, Troubleshoot)
6. **Content Type** — lab or demo
7. **Products & Technologies** — official Red Hat product names
8. **Module Map** — table with module number, title, estimated duration
9. **Difficulty Level** — beginner, intermediate, or advanced
10. **Environment** — what the learner sees when the lab starts, plus automation needs
11. **Infrastructure Requirements** — platform, sizing, AI/MaaS, external services, non-GA products

**Optional sections** (not required by validation):
- **Assessment Strategy** — how success is measured per module. Relevant for Zero-Touch labs with solve/validate buttons. Skip for demos and classic labs without automated checks.

## Infrastructure Requirements

Capture what you know now — guesstimates are fine during intake. Spec refinement fills gaps.

### Base Infrastructure
- Which base CI type: `ocp4-cluster`, `ocp-workloads`, `cloud-vms-base`, or existing CI name
- `cloud_provider`: `cnv` (default), `aws`, `azure`, `aro`, `rosa`, `gcp`, or `google`. CNV unless exception granted. **Automatically set to `aws` when `ai_requirement: gpu` (overwrites existing value).** Azure/ARO/ROSA for cloud-specific deployments. GCP/Google for Google Cloud. Stored in `spec.environment.cloud_provider`.
- Automation approach: Ansible, GitOps (Helm + ArgoCD), or combo

### Cluster Sizing
- Node types and counts with resources. Say "6 workers (8 vCPU, 32GB RAM, 100GB disk)" not just "OpenShift cluster"
- GPU nodes: count + type if applicable
- RHEL nodes: count + sizing if VM-based lab

### Multi-User
- Topology: shared-cluster, per-student, or cnv-pool
- Max concurrent users: required if per-student or cnv-pool

### AI / MaaS
- `ai_requirement`: maas | gpu | none
- `ai_model_tier`: open-source (default, no justification required) | frontier (requires justification)
- `ai_model_name`: specific model if known
- `ai_justification`: required if frontier or gpu — explain why open-source is insufficient
- **AI keyword triggers (Phase 5):** AI, RHOAI, OpenShift AI, MaaS, Granite, InstructLab, Ollama, LLM, inference, model serving
- **Default path:** MaaS + open-source → no justification required. Frontier or GPU → justification required for infra review.

### External Services
- List ALL named external services the lab needs to reach — both during provisioning/deployment and during the student session
- Includes: container registries, package repos, Git hosts, license servers, external APIs, anything automation pulls from at setup time
- "None" is only valid if the environment is fully air-gapped with all content pre-staged
- Vague entries ("internet", "any public IP") → rejected
- Empty list → no additional justification required

### AAP
- Version required if "Ansible Automation Platform" in products

### Non-GA Products
- List non-GA products/versions
- Include access plan: how will access be provided during provisioning?
- Empty list → no additional justification required; non-empty → routes to infra review

Not all fields must be known at intake. "TBD but estimating ~X" is fine.

## Approval Checklist Fields (spec.yaml)

The following are authored during intake and stored in `approval_checklist` in spec.yaml:

| Field | Where | What |
|---|---|---|
| `catalog_gap` | `approval_checklist.content` | Derived from RCARS results — what this design covers that existing catalog items don't |
| `design_overview` | `approval_checklist.content` | Skill-generated 2-3 sentence summary of design.md |
| `module_summaries` | `approval_checklist.content` | Skill-generated [{title, overview}] — one per module |

Auto-computed by Central (not authored):
- `rcars_overlap_pct` — highest relevance_score from RCARS advisor candidates (or null)
- `rcars_top_matches` — top 3 RCARS advisor matches with title, ci_name, url, relevance_score, why_it_fits
- `peak_environments` — max_concurrent_users × topology factor
- `cost_per_run_est` — indicative cost from sizing

## Module Outline Required Sections

Each module outline in `publishing-house/spec/modules/module-NN-*.md` must have:

1. **Brief Overview** — non-empty
2. **Audience and Time** — must include a duration estimate
3. **Learning Objectives** — at least one item
4. **Lab Structure** — table with at least one row
5. **Key Takeaways** — non-empty

## Optional Sections

- Design Principles — pedagogical approach, constraints
- Success Criteria — how to measure effectiveness
- Differentiation — how this differs from existing content (also captured in approval_checklist)

## Quality Checks

### Learning Objectives
- Start with action verbs: Configure, Deploy, Create, Implement, Troubleshoot, Monitor, Scale
- NOT: Understand, Learn, Know, Be familiar with (too vague)
- Each objective should be testable

### Overview
- What the lab or demo is and why it exists (2-3 sentences)
- Followed by a direct description of what participants will do — specific enough to understand at a glance, no interpretation required
- No flowery language or prose

### Module Map
- Each module should be 10-30 minutes
- Total duration should match content type (lab: 1-4 hours, demo: 15-45 minutes)
- Modules should build on each other logically

### Products & Technologies
- Use official Red Hat product names (Red Hat OpenShift, not just OpenShift)
- Include version if relevant
- List upstream projects separately

### Environment
- **Learner view first** — describe what exists when the lab starts
- **Automation scope second** — what automation must provision
- Be specific about cluster requirements

### Assessment Strategy
- Must be explicit per module
- "Trust-based" is acceptable but must be stated clearly
- Prefer verification scripts or visible UI results where possible
