---
name: rhdp-publishing-house:reviewer-helper
description: This skill should be used when the user asks to "review my content", "review module N", "review again", "re-review", "check it again", "technical edit", or "review all". Optional helper that reviews Showroom AsciiDoc content against Red Hat quality standards and Publishing House spec alignment. Not mandatory — authors may skip review or use their own process.
context: main
---

# Reviewer Helper

You perform technical editing by spawning `rhdp-publishing-house:module-reviewer` and adding
Publishing House-specific spec alignment checks. You verify content quality AND
alignment with the approved project spec.

This is an **optional helper**. Authors are never required to run it — content can be marked
complete (via the `rhdp-publishing-house:development` skill) without ever invoking this skill.
Use it whenever the author wants an AI quality pass, whether the content was written by
`rhdp-publishing-house:writer-helper` or by the author themselves.

See @rhdp-publishing-house/skills/reviewer-helper/references/editing-checklist.md for the full editing checklist.
See @rhdp-publishing-house/skills/common/user-interaction.md for how to present multi-option choices to the author.

## Step 1: Determine Scope

Check what the user requested:

- **"edit module N"** / **"review module N"** → review that specific module
- **"edit all" / "review content" / "technical edit"** → review all drafted modules
- **"edit"** with no qualifier → review the next un-reviewed drafted module
- **"review again" / "re-review" / "check it again"** → re-run this same procedure against the
  current state of the file (after manual edits or previous fixes)

**If no modules are drafted:**
> "No drafted modules found. Would you like to write a module first? The `writer-helper` skill can
> help, or you can write the `.adoc` file yourself."

## Step 2: Spawn rhdp-publishing-house:module-reviewer Agent

For each module to review, spawn the agent via Task tool:

    Task tool:
      subagent_type: rhdp-publishing-house:module-reviewer
      prompt: |
        MODULE_FILE: content/modules/ROOT/pages/<module-filename>.adoc
        CONTENT_TYPE: <workshop|demo>
        LAB_TYPE: <ocp|rhel|vm|ai>
        SHARED_CONTEXT: <JSON with module_order, defined_attributes, cross-module refs>
        REPO_PATH: <absolute repo path>

## Step 3: Run Spec Alignment Checks

After `rhdp-publishing-house:module-reviewer` completes, run PH-specific checks.

Read the module outline from `publishing-house/spec/modules/module-NN-*.md`.
Read the generated content from the content directory.
Read `publishing-house/spec/design.md` for project-level context.

### SA-1: Outline Coverage
### SA-2: Learning Objectives Match
### SA-3: Duration Alignment
### SA-4: Cross-Module Consistency (only when reviewing multiple modules)
### RS-1: Product Name Accuracy
### RS-2: Version Consistency

See @rhdp-publishing-house/skills/reviewer-helper/references/editing-checklist.md for details.

## Step 4: Produce Review Report

Present a summary to the author:

> **Module N — Review Complete**
>
> **Findings:** [count] issues found
> [List CRITICAL and HIGH findings]
>
> **These findings are directions, not mandatory fixes.** Review them and fix what makes sense
> for your content — some may not apply to your specific lab.
>
> **What would you like to do?**
> 1. Edit the file yourself, then type **1** again — I'll re-check your changes
> 2. Ask me to fix specific items — tell me what to change and I'll update the file
> 3. Done — go back to the **development** skill dashboard to mark the module complete

Write the full review report to `publishing-house/reviews/editing-review-module-NN.md`.

## Step 5: Fix Loop

Enter fix loop based on autonomy:

- **Supervised:** Present report, ask which issue to fix first.
- **Semi:** Auto-fix MEDIUM/LOW, present CRITICAL/HIGH for decision.
- **Full:** Auto-fix all with clear fixes, present judgment calls.

After fixes, re-run spec alignment checks to verify.

## What You Do NOT Do

- Do not write new content — that is `rhdp-publishing-house:writer-helper`'s responsibility
- Do not modify the module outlines without user confirmation
- Do not advance the lifecycle phase
- Do not create or modify scaffold files — scaffolding is the `rhdp-publishing-house:development` skill's domain
- Do not mark modules or the showroom complete, and do not submit to Central — that is the
  `rhdp-publishing-house:development` skill's job
