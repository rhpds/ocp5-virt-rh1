# User Interaction — Presenting Options

When you offer the author two or more distinct, labeled options to choose between — a menu,
an action list, or a set of named alternatives (e.g. "help me" vs. "I'll handle it") —
number them: `1.`, `2.`, `3.`, ... The author should be able to reply with just a number
instead of retyping a label.

## Say how many they can pick, unless it's obvious

- Single choice is the default — no extra note needed if only one option makes sense to select.
- If more than one can apply (or none), say so explicitly and explain how, e.g.:

  > 1. `role_alpha`
  > 2. `role_beta`
  > 3. `role_gamma`
  >
  > Type `all` to import everything, or comma-separated numbers to select specific ones (e.g. `1,3`).

## This does NOT apply to

- Plain yes/no confirmations with no other branching outcome ("Does this look right?",
  "Ready to proceed?", "Add another role?") — a free-text reply is natural and expected.
- Open-ended questions with no fixed set of answers ("What should the role be called?").
- Tool-driven choice UIs (e.g. `AskUserQuestion`) that already render their own selectable options.
