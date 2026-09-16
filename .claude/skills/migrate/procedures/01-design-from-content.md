# Design Doc from Content

Generate a complete design.md by reverse-engineering the imported content/ folder.

## Read Content

1. Read `publishing-house/spec/design.md` — this is the template with placeholder sections
2. Read `publishing-house/spec.yaml` inline comments to understand valid values for constrained fields
3. Read all module pages from `content/modules/ROOT/pages/` — these are the actual lab content
4. Read `content/modules/ROOT/nav.adoc` — module ordering
5. Read `site.yml` — project title and metadata

## Generate Each Section

For each section in design.md, replace the placeholder text with content derived from the existing modules:

1. **H1 Title** — use the title from site.yml or spec.yaml project.description
2. **Overview** — synthesize from the overall content: what the lab/demo is, why it exists, what participants will do. Direct, no flowery prose.
3. **Target Audience** — infer from the content difficulty, prerequisites mentioned in modules, and products used
4. **Prerequisites** — extract from any prerequisites sections in the content, or infer from what the modules assume the learner knows
5. **Learning Objectives** — derive from what each module teaches. Use action verbs from the policy's `action_verbs_valid` list (Configure, Deploy, Create, Implement, etc.). Scale to duration: up to 3 objectives per 45 minutes.
6. **Content Type** — read from `project.content_type` in spec.yaml
7. **Products & Technologies** — extract all Red Hat product names, operators, and tools mentioned in the content. Validate against the policy product list.
8. **Module Map** — create a table from nav.adoc entries with estimated durations based on content length and complexity
9. **Difficulty Level** — infer from content complexity (beginner, intermediate, advanced)
10. **Environment** — describe what the learner sees when the lab starts, based on what the content assumes is pre-provisioned
11. **Infrastructure Requirements** — leave as "TBD — confirmed in infrastructure phase" (same as fresh intake)

**Do NOT fill in the Infrastructure Requirements section.** Leave every field as `TBD — confirmed in infrastructure phase`. Infrastructure is determined in Phase 5.

## Present for Review

Present the design doc to the author:

> "Here's the design doc I've generated from your existing content. Review it and let me know if anything needs changing."

**Wait for explicit approval.** Do NOT proceed until the author confirms.

- **If feedback** → update design.md, re-present
- **If approved** → write and continue

## Write Point

Write design.md and update corresponding spec.yaml fields:
- `spec.title`
- `spec.audience`
- `spec.duration_hours`
- `spec.learning_objectives`
- `spec.modules` — with stable IDs (module-01, module-02, etc.)
- `approval_checklist.content.prerequisites_verifiable`

```bash
git add publishing-house/spec/design.md publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "feat: design doc generated from imported content" 2>/dev/null || true
```

## Inline Structure Check

After writing design.md, validate against the spec guidelines (`@rhdp-publishing-house/skills/intake/references/spec-guidelines.md`):

- All required sections present (11 sections + descriptive H1 title)
- Learning objectives use valid action verbs
- No unfilled template placeholders
- Module durations in 10-60 minute range
- Module Map table exists with at least one row

> "Quick structure check: [results]. Want to fix these now, or proceed?"

Proceed to Phase 3 (RCARS vetting).
