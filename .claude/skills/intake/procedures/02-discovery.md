# Discovery

Phase 1 of the intake flow. Capture the author's idea through conversation.

## Entry Paths

### Path A — "Build on the description"

**Check for pre-intake data first**: If `project.preintake_outline` exists in spec.yaml, the author provided pre-intake information during the pre-intake workflow phase. Present what was captured:

> "I see you provided pre-intake information during onboarding. Here's what I have:
> - **Asset Title**: {spec.title}
> - **Audience**: {spec.audience}
> - **Content Type**: {project.content_type}
> - **Duration**: {spec.duration_hours} hours
> - **Learning Objectives**: {spec.learning_objectives}
>
> Would you like to build on this, or would you like to revise any of it?"

If no pre-intake data exists, read `project.description` from spec.yaml and present it:

> "Here's the description you provided: *'{description}'* — would you like to build on this, or would you like to take it in a different direction?"

Accept whatever the author provides. Extract what you can from their response — goal,
audience, products, content type, duration. Then ask targeted follow-ups for what's missing,
one at a time.

**Discover, don't interrogate.** The author's words are the spec. You are the scribe, not
the author. If something is unclear, ask — don't fill it in.

### Path B — "I have something written up elsewhere"

The author has requirements in an existing document — Google Doc, meeting notes, Jira issue,
rough outline — that is NOT in the Publishing House project repo. It needs to be converted.

1. Read whatever they provide (pasted content, file path, URL, or "paste the Jira requirements")
2. Extract the data points below into PH format
3. Present what was found: "Here's what I extracted — does this look right?"
4. Ask about gaps — missing data points only

### Path C — "The design is already in the repo"

The author already filled in `publishing-house/spec/design.md` and possibly the module
outlines directly in the project repo. They don't need an interview — they need validation
and gap-fill.

1. Read `publishing-house/spec/design.md` and `publishing-house/spec.yaml`
2. Check the four required data points for Phase 2: **goal** (spec.title non-empty), **audience** (spec.audience non-empty), **products** (Products & Technologies section filled in design.md), **content type** (project.content_type set)
3. If any of the four are missing or still placeholders → ask about those specific gaps
4. If all four are present → confirm: "Your design looks populated. Let me check for any gaps in infrastructure and approval fields."
5. Skip to Phase 5 (infrastructure confirmation) or Phase 6 (finalize) depending on what's still empty

## Data Points to Capture

These are the things you need to learn during discovery. Do NOT ask them as a rigid
question list — extract them naturally from the conversation and follow up on gaps.

**Important**: Many of these fields may already be populated from pre-intake ProForma data (Step 4e). Check spec.yaml before asking — skip any field that already has a value.

- **Goal** — What will someone be able to DO after completing this? Concrete, measurable. (→ `spec.title`)
- **Target audience** — Who is this for? Role, experience level, what they already know. (→ `spec.audience`)
- **Products and technologies** — Which Red Hat products are involved? Validate names against the policy's product list (with aliases). If a name isn't recognized, flag it: "I don't see that in the product list — the closest match is [X]. Is that what you mean?"
- **Content type** — Lab (hands-on) or demo (presenter-led). Skip if `project.content_type` is already set in spec.yaml.
- **Showroom type** — Classic or zero-touch. Skip if `project.showroom_type` is already set.
- **Duration estimate** — How long should this take end to end? (→ `spec.duration_hours`)
- **Reference material** — Do they have existing docs, recorded demos, blog posts, architecture diagrams? Note these for Phase 2.
- **Content outline** — If `project.preintake_outline` exists, review it with the author and use it to inform module structure in Phase 2.

## Behavioral Notes

- Ask follow-up questions when answers are vague. "Teach OpenShift" is not a goal — probe for specifics.
- Scale learning objectives to duration. Guideline: up to 3 objectives per 45 minutes of content. A 45-minute demo might have 2-3; a 2-hour lab might have 6-8; a 4-hour lab could have 10+. Too few for the duration means objectives are too broad; too many means they're too granular.
- If the author describes module content in detail, capture it — it feeds directly into Phase 2.
- Use what the author gives you. Don't substitute your own ideas for theirs.
- If one answer also covers a later data point, mark it as captured and don't re-ask.

## Write Point

**Minimum to proceed:** You need at least **goal** (→ `spec.title`), **audience** (→ `spec.audience`), **products** (→ design.md Products section), and **content type** (→ `project.content_type`). Without these four, you cannot generate a design doc in Phase 2. If any are missing after the conversation, ask specifically.

At the end of this phase, write all captured discovery fields to `publishing-house/spec.yaml`:
- `spec.title` (derived from the goal)
- `spec.audience`
- `spec.duration_hours`
- `project.content_type` (if not pre-set)
- `project.showroom_type` (if not pre-set)

```bash
git add publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "feat: phase 1 — discovery fields captured" 2>/dev/null || true
```

Proceed to Phase 2 (design generation).
