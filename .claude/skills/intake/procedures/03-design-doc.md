# Design Doc

Phase 2 of the intake flow. Produce a complete, validated design.md.

## Generate the Design Doc

Read `publishing-house/spec/design.md` from the project repo. This is the template — it has
placeholder sections that you will fill in using what you learned in Phase 1 (discovery).

Read `publishing-house/spec.yaml` inline comments to understand valid values for constrained
fields (e.g., `# cnv | aws | azure | aro | rosa | gcp | google` for cloud_provider). Use these as guidance when filling
in the corresponding design.md sections.

For each section in design.md:
1. Replace the placeholder text with real content from the discovery conversation
2. Keep the section heading exactly as-is — the validation engine checks for these
3. If you don't have enough information for a section, leave a clear note: "TBD — [what's needed]"

**Do NOT fill in the Infrastructure Requirements section.** Leave every field in that section as `TBD — confirmed in infrastructure phase`. Do not apply defaults (e.g., do not default cloud provider to CNV). Infrastructure is determined in Phase 5 after the author's products and platform are fully understood.

**Propose module structure.** Based on the discovery conversation, propose a Module Map table
with titles and estimated durations. Explain why you structured it this way. The author may
adjust — that's expected.

**Scale learning objectives to duration.** Use the ratio of up to 3 objectives per 45 minutes of content. Do not default to 3–4 regardless of lab length — a 2-hour lab should have 6–8 objectives; a 4-hour lab 10 or more. Too few means objectives are too broad; too many means they're too granular.

## Present for Review

Present the design doc to the author — the design.md content only, NOT spec.yaml:

> "Here's the design doc I've drafted. Review it and let me know if anything needs changing.
> You can also edit `publishing-house/spec/design.md` directly in your editor."

**Wait for explicit approval.** Do NOT generate module outlines until the author says the
design looks good.

- **If feedback** → update design.md, re-present
- **If "I already filled this out"** → read the existing design.md, validate it has content
  (not just placeholders), and proceed
- **If approved** → write and continue

## Re-intake After Rejection

When called after the rejection handler (00-rejection-handler.md), design.md already exists.
Do NOT regenerate from scratch.

1. Read `publishing-house/spec.yaml` to understand what changed during rejection resolution
2. Read the existing `publishing-house/spec/design.md`
3. Identify sections affected by the changes — rewrite them coherently
4. Present the updated sections to the author for confirmation

## Write Point

Write design.md and update corresponding spec.yaml fields.

**Set `prerequisites_verifiable` in spec.yaml** based on the prerequisites section: if verification is automated (solve/validate buttons, scripts), set `true`; if trust-based (no automated check), set `false`. This must be written now — do not leave it null.

```yaml
approval_checklist:
  content:
    prerequisites_verifiable: false   # or true if automated
```

When writing `spec.modules`, assign each module a stable `id` — `module-01`, `module-02`, etc.
These IDs are permanent: if modules are later reordered, renamed, or removed, the ID stays
with the content it was assigned to. Set `status: not_started` on every new module. Example:

```yaml
spec:
  modules:
  - id: module-01
    title: "Getting Started with OpenShift"
    duration_min: 30
    status: not_started
  - id: module-02
    title: "Deploying Applications"
    duration_min: 45
    status: not_started
```

```bash
git add publishing-house/spec/design.md publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "feat: phase 2 — design doc generated" 2>/dev/null || true
```

## Inline Structure Check

After writing design.md, run a quick validation against the spec guidelines
(`@rhdp-publishing-house/skills/intake/references/spec-guidelines.md`):

- All required sections present (11 sections + descriptive H1 title)
- Learning objectives use valid action verbs (from policy)
- No unfilled template placeholders (`[placeholder text]` markers)
- Module durations in 10-60 minute range
- Module Map table exists with at least one row

**This check is non-blocking.** Show the results. If there are issues, help the author fix
them before proceeding. But do not gate the flow — the hard gate is at submission (Phase 6).

> "Quick structure check on your design doc: [results]. Want to fix these now, or proceed?"

Proceed to Phase 3 (RCARS vetting).
