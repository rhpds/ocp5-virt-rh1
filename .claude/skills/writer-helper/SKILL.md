---
name: rhdp-publishing-house:writer-helper
description: This skill should be used when the user asks to "write a module", "draft content", "start writing", "write module N", "write all", "generate the index", "generate the conclusion", or "continue writing". Optional helper that generates Showroom AsciiDoc content from a project's spec/design/module outlines. Not mandatory — authors may write content themselves instead.
context: main
---

# Writer Helper

You write Showroom AsciiDoc content by spawning the `rhdp-publishing-house:module-writing-helper` agent
with context from the project's spec files. You do NOT write AsciiDoc directly.

This is an **optional helper**. Authors are never required to use it — they may write `.adoc` files
themselves with any tool they like. When you finish writing something, you hand control back to the
author (and to the `rhdp-publishing-house:development` skill for status tracking and submission).

See @rhdp-publishing-house/skills/writer-helper/references/writing-standards.md for writing standards.
See @rhdp-publishing-house/skills/common/user-interaction.md for how to present multi-option choices to the author.

## CRITICAL CONSTRAINT — Content Only, No Scaffolding

You write `.adoc` module files into an already-scaffolded showroom repo.
You MUST NOT create or modify scaffold files (`site.yml`, `ui-config.yml`, `antora.yml`, directory structure).
Scaffolding is handled by `rhdp-publishing-house:development` (via `config-helper`/`config-reviewer`).
If the repo isn't scaffolded yet, tell the author to run the `rhdp-publishing-house:development` skill first.

## Step 1: Determine Which Module to Write

Check what the user requested:

- If user said "write module N" → write that specific module
- If user said "write all" or "start writing" → write all pending modules sequentially
- **Never write modules in parallel.** Each module depends on the previous one for story continuity.

## Step 2: Read Project Context

Read three files from the author's project repo:

1. **`publishing-house/spec.yaml`** — machine-readable metadata: environment (ocp_version, topology, cloud_provider), module list, audience, duration
2. **`publishing-house/spec/design.md`** — human-readable narrative: overview, audience, prerequisites, products, business scenario
3. **`publishing-house/spec/modules/module-NN-*.md`** — detailed step-by-step outline for the target module

Build a combined context object from all three sources.

## Step 3: Check Module Status (Sequential Enforcement)

Before spawning any agent, check `spec.yaml` module statuses:

- `not_started` → eligible to write
- `in_progress` → resume this module (started but not finished)
- `complete` → skip; move to next module

**Sequential rule:** Module N CANNOT start until modules 1 through N-1 are ALL `complete`.

Present a plan before spawning:
> "Here's what I'll write for module N: [summary of outline]. Ready to proceed?"

Wait for user approval — never auto-generate.

## Step 3b: Mark Module In Progress

After the author approves, update `spec.yaml` before spawning the writing agent:

1. Change the module's `status: not_started` to `status: in_progress`
2. Commit and push:
   ```bash
   git add publishing-house/spec.yaml
   git commit -m "feat: start module N — [title]"
   git push
   ```

This is bookkeeping only — it does not mark the module complete. Only the
`rhdp-publishing-house:development` skill sets `status: complete`.

## Step 4: Spawn rhdp-publishing-house:module-writing-helper Agent

For each module, spawn the agent via Task tool:

Derive the `.adoc` filename from the outline filename in `publishing-house/spec/modules/`:
replace `.md` with `.adoc`. Example: outline `module-01-pipeline-setup.md` → `content/modules/ROOT/pages/module-01-pipeline-setup.adoc`.

    Task tool:
      subagent_type: rhdp-publishing-house:module-writing-helper
      prompt: |
        TARGET_FILE: content/modules/ROOT/pages/<outline-name>.adoc
        FILE_TYPE: module
        FULL_SPEC: <JSON from spec.yaml + design.md + module outline>
        LAB_TYPE: <ocp|rhel|vm|ai>
        CONTENT_TYPE: <workshop|demo>
        SHOWROOM_TYPE: <classic|zero_touch from project.showroom_type in spec.yaml>
        REPO_PATH: <absolute repo path>

One agent per module, run sequentially.

## Step 5: Content Commit and Verification

After the writing agent finishes:

1. Commit and push the written content immediately:
   ```bash
   git add content/
   git commit -m "feat: write module N — [title]"
   git push
   ```
2. Verify the generated file exists in `content/modules/ROOT/pages/`
3. Check that `content/modules/ROOT/nav.adoc` includes the new module
4. Scan for placeholders and open items:
   - Image references (`image::`) where the image file doesn't exist in `content/modules/ROOT/assets/images/`
   - Placeholder text like `TODO`, `FIXME`, `[placeholder]`, `TBD`
   - Diagram references without corresponding files
5. Collect all open items into a list

## Step 5c: Present Results and Hand Back to the Author

Present a summary to the author:

> **Module N — Writing Complete**
>
> **Open Items:**
> - [List any missing images, placeholders, TODOs from Step 5]
>
> **Please review the generated file:**
> `content/modules/ROOT/pages/[filename].adoc`
>
> Open it, read through the content, and check that it matches what you expect.
> The AI handled ~80% — your review covers the rest: accuracy, tone, missing context, and any items above.
>
> **What would you like to do?**
> 1. Looks good — done with this module
> 2. Edit the file yourself, then type 1 when ready
> 3. Fix specific items — tell me what to change and I'll update the file
> 4. Run AI quality pass (`rhdp-publishing-house:reviewer-helper`)

- **1** → return to the calling skill (development will mark the module complete).
- **2** → wait for the author to finish editing and type 1.
- **3** → apply fixes, then re-present this menu.
- **4** → dispatch to `rhdp-publishing-house:reviewer-helper`, then re-present this menu.

**STOP HERE.** Do NOT mark the module complete yourself. Do NOT proceed to the next module without
being asked. Marking complete and submitting to Central are the `rhdp-publishing-house:development`
skill's job, not yours.

## What You Do NOT Do

- **NEVER write AsciiDoc files directly** — always spawn rhdp-publishing-house:module-writing-helper
- **NEVER create or modify scaffold files** (`site.yml`, `ui-config.yml`, `antora.yml`, `nav.adoc` structure)
- **NEVER write modules in parallel**
- **NEVER auto-run a reviewer** — reviewing is opt-in via `rhdp-publishing-house:reviewer-helper`
- **NEVER mark a module or the showroom complete, and NEVER submit to Central** — that is the
  `rhdp-publishing-house:development` skill's job

## Step 6: Generate Index and Conclusion (After All Modules Complete)

When all modules show `status: complete` in `spec.yaml`, generate the two capstone files:
**`index.adoc`** — learner-facing introduction
**`conclusion.adoc`** — recap of learning objectives and next steps

### Step 6a: Check Prerequisites

Before generating index and conclusion:

1. Verify all modules are complete: check `spec.yaml` — every module must show `status: complete`
2. Identify the target paths in `content/modules/ROOT/pages/`:
   - `index.adoc`
   - `conclusion.adoc`
3. Confirm `spec.yaml` module list is intact — you'll need the full list for the conclusion's "What You've Learned" recap

Present a plan before proceeding:
> "All modules are complete. I'll now generate index.adoc and conclusion.adoc to finalize your showroom."

Wait for approval — never auto-generate.

### Step 6b: Generate Index (FILE_TYPE: index)

Spawn the module-writing-helper agent:

    Task tool:
      subagent_type: rhdp-publishing-house:module-writing-helper
      prompt: |
        TARGET_FILE: content/modules/ROOT/pages/index.adoc
        FILE_TYPE: index
        FULL_SPEC: <JSON from spec.yaml + design.md>
        LAB_TYPE: <ocp|rhel|vm|ai>
        CONTENT_TYPE: <workshop|demo>
        SHOWROOM_TYPE: <classic|zero_touch from project.showroom_type in spec.yaml>
        REPO_PATH: <absolute repo path>

Wait for the agent to complete. Verify the file exists.

Commit immediately:
```bash
git add content/modules/ROOT/pages/index.adoc
git commit -m "feat: generate index.adoc"
```

Present to the author:
> **Index — Writing Complete**
>
> Please review: `content/modules/ROOT/pages/index.adoc`
>
> **Options:**
> 1. Accept — move on to conclusion
> 2. Fix specific items — tell me what to change
> 3. AI review — I'll run `rhdp-publishing-house:reviewer-helper` on it

**STOP.** Wait for the author to select an option before moving to conclusion.

### Step 6c: Generate Conclusion (FILE_TYPE: conclusion)

Only after the author approves index.

Spawn the module-writing-helper agent:

    Task tool:
      subagent_type: rhdp-publishing-house:module-writing-helper
      prompt: |
        TARGET_FILE: content/modules/ROOT/pages/conclusion.adoc
        FILE_TYPE: conclusion
        FULL_SPEC: <JSON from spec.yaml + design.md with complete module list>
        LAB_TYPE: <ocp|rhel|vm|ai>
        CONTENT_TYPE: <workshop|demo>
        SHOWROOM_TYPE: <classic|zero_touch from project.showroom_type in spec.yaml>
        REPO_PATH: <absolute repo path>

Wait for the agent to complete. Verify the file exists.

Commit immediately:
```bash
git add content/modules/ROOT/pages/conclusion.adoc
git commit -m "feat: generate conclusion.adoc"
```

Present to the author:
> **Conclusion — Writing Complete**
>
> Check that all learning objectives from your modules are captured in "What You've Learned".
>
> Please review: `content/modules/ROOT/pages/conclusion.adoc`
>
> **Options:**
> 1. Accept — finalize conclusion
> 2. Fix specific items — tell me what to change
> 3. AI review — I'll run `rhdp-publishing-house:reviewer-helper` on it

**STOP.** Wait for the author to select an option.

### Step 6d: Verify Navigation and Structure

After both index and conclusion are approved:

1. Check `content/modules/ROOT/nav.adoc` includes both files in correct order:
   - First entry: `index.adoc`
   - Last entry: `conclusion.adoc`
2. Verify no placeholder text or `TODO` markers in either file
3. Check that all learning objectives from modules are consolidated in conclusion's "What You've Learned" section

Then tell the author:
> "Index and conclusion are written and reviewed. If you want a final config/tab check, ask the
> **development** skill to run its config-reviewer. When you're ready, tell the **development**
> skill your content is complete and say 'submit to central' — it handles marking the showroom
> complete and submitting."

Do NOT skip Step 6 once all modules are done. Index and conclusion are mandatory for a complete showroom.
Do NOT mark the showroom complete or submit to Central yourself — hand back to `development`.
