# Infrastructure Confirmation

Phase 5 of the intake flow. Capture infrastructure requirements as a single confirm-or-adjust interaction.

## Check Pre-populated Infrastructure Fields

**Read the Jira epic "Environment Configuration" section first.** These fields are pre-populated from the onboarding form:
- **Platform** (`ocp` or `rhel-vms`)
- **Cloud Provider** (`cnv`, `aws`, `rosa`, `aro`)
- **Cluster Type** (`sno`, `multinode`) — only if platform=ocp
- **OCP Version** (e.g., `4.20`, `4.21`) — only if platform=ocp
- **RHEL Version** (e.g., `8`, `9`, `10`) — only if platform=rhel-vms

These values serve as **starting points**. Present them to the author for confirmation or adjustment.

If the epic does not have these fields (legacy flow), determine the platform from the products discussed in discovery:

| Signal | Platform |
|--------|----------|
| Products include OpenShift, OCP, OCP Virt, RHOAI, or any OCP operator | `platform: ocp` |
| Products are AAP, RHEL, Satellite, or other non-OCP products | `platform: rhel-vms` |
| Mixed (e.g., AAP + OpenShift) | `platform: ocp` (OCP is the infrastructure base) |

## Derive Defaults

Based on platform and products. **Use pre-populated values from the epic as starting points.**

### For `platform: ocp`

| Signal | Default |
|--------|---------|
| Epic has Cluster Type | Use epic value, confirm with author |
| Epic has OCP Version | Use epic value, confirm with author |
| Products include OCP Virtualization/CNV | `cluster_type: multinode` if not already set |
| Simple OCP lab | `cluster_type: sno` if single-user, `multinode` otherwise (if not already set) |
| OCP version not specified | `ocp_version: "4.20"` (minimum) |

### For `platform: rhel-vms`

**Read RHEL Version from the epic "Environment Configuration" section.** This value MUST be used for the `version` field of all VMs in `vms_per_student`.

Propose per-student VM roles based on the products. For example:
- AAP lab → 1 AAP controller (8 vCPU, 32GB), 2 RHEL managed nodes (2 vCPU, 8GB each)
- AAP + Windows → add 1 Windows Server node (4 vCPU, 8GB)
- AAP + EDA → add EDA controller resources or increase AAP controller sizing

**Each VM entry MUST include the RHEL version from the epic:**
```yaml
vms_per_student:
  - role: aap-controller
    count: 1
    cpu: 8
    ram_gb: 32
    disk_gb: 100
    os: rhel
    version: "9"  # From epic "RHEL Version: 9"
  - role: managed-node
    count: 2
    cpu: 2
    ram_gb: 8
    disk_gb: 50
    os: rhel
    version: "9"  # Same version for all RHEL VMs
```

Do NOT propose OCP fields (ocp_version, cluster_type, control_plane_*, worker_*) for
RHEL-based labs. Those fields are irrelevant.

### Common defaults (both platforms)

**AI Requirement Mapping from Pre-Intake Form:**

The workflow data contains these AI-related fields from the pre-intake submission:
- `aiRelated` (boolean) - whether the project is AI-related
- `canUseMaas` (boolean) - whether MaaS is sufficient (defaults to `true`)
- `maasModels` (string) - which MaaS models are needed
- `gpuJustification` (string) - why direct GPU access is required

**Map workflow data to spec.yaml `spec.environment.*` fields:**

| Workflow Data | spec.environment Mapping |
|---------------|-------------------------|
| `aiRelated: false` | `ai_requirement: none` |
| `aiRelated: true` AND `canUseMaas: true` | `ai_requirement: maas`<br/>`ai_model_tier: open-source` (default)<br/>`ai_model_name: <maasModels value>` |
| `aiRelated: true` AND `canUseMaas: false` | `ai_requirement: gpu`<br/>`ai_justification: <gpuJustification value>`<br/>`cloud_provider: aws` (overwrites existing) |

| Signal | Default |
|--------|---------|
| Products include AI keywords (no pre-intake data) | `ai_requirement: maas`, `ai_model_tier: open-source` |
| AI requirement is GPU | `cloud_provider: aws` (overwrites existing value) |
| No specific cloud reason | `cloud_provider: cnv` (platform default) |
| Azure-based deployments | `cloud_provider: azure` |
| ARO (Azure Red Hat OpenShift) | `cloud_provider: aro` |
| ROSA (Red Hat OpenShift Service on AWS) | `cloud_provider: rosa` |
| No topology discussed | `topology: shared-cluster` |

## Present as One Profile

Present a complete infrastructure profile for confirmation — not individual questions.
Only include fields relevant to the detected platform.

**For OCP labs:**
> "Based on your products, here's the infrastructure profile:
>
> - **Platform:** OCP
> - **Cloud provider:** CNV
> - **Cluster type:** Multinode, 6 workers (8 vCPU, 32GB RAM, 100GB disk)
> - **OCP version:** 4.20
> - **Topology:** Per-student
>
> Does this look right, or should I adjust anything?"

**For RHEL-based labs:**
> "Based on your products, here's the infrastructure profile:
>
> - **Platform:** RHEL VMs (provisioned via CNV)
> - **Per student:**
>   - 1 AAP controller (8 vCPU, 32GB RAM)
>   - 2 RHEL managed nodes (2 vCPU, 8GB RAM each)
>   - 1 Windows Server (4 vCPU, 8GB RAM)
> - **Topology:** Per-student
> - **AAP version:** 2.5
>
> Does this look right, or should I adjust anything?"

## Conditional Follow-ups

Only ask these if triggered — do not ask them by default:

- **AI/MaaS:** Only if products include AI keywords. Default to MaaS + open-source.
  If author chooses frontier model or GPU: "Why is an open-source model insufficient?"
- **AAP version:** Only if AAP is in the products list.
- **Non-GA products:** Only if any product is labeled beta, tech preview, or early access.
  If non-empty: "How will access be provided during provisioning?"
- **Concurrent users:** Only if topology is shared-cluster. Per-student and cnv-pool
  topologies give each learner their own environment — the number of simultaneous users
  is an operational decision made at scheduling time, not during intake. Do NOT ask the
  author how many people will be in a room or at an event.
- **External services:** Ask explicitly. This covers ALL external services the lab environment
  needs to reach — both during provisioning/deployment and during the student session. Do NOT
  let the author answer "none" based only on what the student sees. Prompt them to consider:
  container registries, package repos, license servers, external APIs, Git hosts, anything
  the automation pulls from during setup. "None" is only correct if the environment is fully
  air-gapped with all content pre-staged. If the author is unsure, list the likely candidates
  based on their products and leave them to confirm.

## Automation Approach

**Ask the author which automation approach they plan to use:**

> "What automation approach will you use for provisioning and deployment?"
>
> - **Ansible** — AgnosticD roles and workloads
> - **GitOps** — Helm charts + ArgoCD
> - **Both** — Hybrid approach (AgnosticD for base infrastructure, GitOps for application layer)

**Map the response to `project.automation_type` in spec.yaml:**
- Ansible → `ansible`
- GitOps → `gitops`
- Both → `both`

**Default recommendation:**
- Most labs and demos → `ansible` (standard RHDP pattern)
- Cloud-native application demos → `gitops`
- Complex multi-layer environments → `both`

## Confirmation Required

**After the author provides adjustments, present the updated profile and wait for
explicit confirmation before proceeding.** Do NOT silently apply changes and move on.

> "Here's the updated infrastructure profile: [show updated fields].
> Does this look good to move forward?"

Only proceed to Phase 6 after the author confirms.

## Rules

- Do NOT fabricate explanations for why a field is relevant when it isn't.
- If the author challenges a field, remove it or leave it blank. Do not invent justifications.
- Only propose fields that are directly relevant to the products and platform.
- If you're unsure whether a field applies, leave it for the infra reviewer.
- Do NOT propose OCP fields (ocp_version, cluster_type, worker_*) for RHEL-based labs.

## Write Point

Write all infrastructure fields to spec.yaml (platform-appropriate fields only) and
update the Infrastructure Requirements section of design.md:

```bash
git add publishing-house/spec.yaml publishing-house/spec/design.md
git diff --cached --quiet || git commit -m "feat: phase 5 — infrastructure confirmed" 2>/dev/null || true
```

Proceed to Phase 6 (finalize + submit).
