# RCARS Vetting

Phase 3 of the intake flow. Validate the design against the existing RHDP catalog.

**If offline → skip this phase entirely with a warning:**
> "RCARS vetting is unavailable offline. This will run when you submit for review."

## Query RCARS

Call Central's RCARS advisor with the project slug:

```bash
python publishing-house/tools/ph-rcars.py submit "QUERY"
```

Build the QUERY from design.md — summarize the project in one sentence using the title,
products, audience, and key learning objectives:

`"A [audience] [content_type] covering [products] that teaches [key objectives from design.md]"`

Extract `job_id` from the output. If empty or failed → skip RCARS with a note:
> "RCARS advisor query failed. Proceeding without vetting — this will run at submission."

## Poll for Results

```bash
python publishing-house/tools/ph-rcars.py poll JOB_ID
```

If `status` is `running` or `queued`, wait 5 seconds and poll again. Keep polling every
5 seconds for up to 90 seconds total. The advisor typically takes 10–20 seconds but can
be slower under load.

## Present Results

**If advisor returns candidates** (status=complete, candidates non-empty):

Present the results as information, not a challenge. The author is not required to justify
their lab's existence. The purpose is awareness — they should know what's already in the
catalog so they can position their design accordingly.

> "I checked the RHDP catalog against your design. Here's what exists:
>
> 1. **[display_name]** ([relevance_score]% relevance)
>    *Similar because:* [why_it_fits — first sentence]
>    *Gap:* [caveats — first sentence]
>
> 2. **[display_name]** ([relevance_score]% relevance)
>    *Similar because:* [why_it_fits — first sentence]
>
> **Summary:** [1-2 sentence synthesis of what this design covers that the existing
> catalog items don't. Derive this from the gaps and caveats in the results.]
>
> Do you want to adjust anything in the design based on this?"

Do NOT ask "how does your lab differ" or "why should this exist." Do NOT present a
"differentiation statement" for the author to review or confirm — the catalog gap summary
is recorded internally in spec.yaml. The author's only decision here is whether the RCARS
findings change anything about the design. Do NOT reference "the advisor" as if it's a
person — RCARS is a catalog search tool, not a reviewer.

**If no candidates** (empty list or all relevance_score < 50):

> "I checked the RHDP catalog — no close matches found. This looks like new territory.
> Moving on to module outlines."

No question needed — the differentiation is self-evident if nothing similar exists.

**If advisor failed or timed out:**

> "RCARS advisor wasn't available this run. We'll skip vetting for now — this will
> run again at submission. Moving on to module outlines."

Do NOT ask the author to explain differentiation when RCARS fails. That's a platform
problem, not the author's problem. The differentiation field will be populated when
RCARS is available (either in a later session or at submission).

## Handle Adjustments

If the author wants to adjust the design based on RCARS findings:
1. Update `publishing-house/spec/design.md` with the changes
2. Re-run the inline structure check from Phase 2
3. The author can also edit design.md directly and commit

If no adjustments → proceed.

## Write Point

The `poll` command automatically enriches each candidate with its workload mappings
(AgnosticD roles/collections) from the RCARS catalog. Candidates with AgD v2 workloads
will include a `workloads` array; others will not.

Write RCARS results, workloads, and differentiation to spec.yaml:

```yaml
approval_checklist:
  content:
    rcars_overlap_pct: [highest relevance_score or null]
    rcars_top_matches:
      - title: "[display_name]"
        ci_name: "[ci_name]"
        url: "https://catalog.demo.redhat.com/catalog?item=[ci_name]"
        relevance_score: [score]
        why_it_fits: "[why_it_fits]"
        workloads:
          - role: "[workload_role]"
            collection: "[workload_collection]"
    catalog_gap: "[derived from RCARS results — summarize what this design covers that existing catalog items don't]"
```

```bash
git add publishing-house/spec.yaml publishing-house/spec/design.md
git diff --cached --quiet || git commit -m "feat: phase 3 — RCARS vetting complete" 2>/dev/null || true
```

Proceed to Phase 4 (module outlines).
