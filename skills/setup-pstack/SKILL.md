---
name: setup-pstack
description: Configure which model tiers pstack uses per role and at what reasoning budget. Writes an always-applied rule that overrides the skill defaults. Use for /setup-pstack, "configure pstack models", "pstack budget", or changing pstack's model choices.
---

# Setup pstack

Write `~/.gemini/config/rules/pstack-models.md`, an always-applied rule that sets pstack's model tier per role.

## Steps

### 1. Detect available model tiers

Antigravity's `invoke_subagent` accepts four `Model` tiers: `pro`, `flash`, `flash_lite`, and `inherit`. You can also run `agy models` to inspect the underlying models available in the current session. The aliases `inherit-parent` and `auto` are always valid and map to `Model: "inherit"`.

### 2. Load current state

The default role-to-model mapping is the rule shape shown in step 5 below. If `~/.gemini/config/rules/pstack-models.md` already exists, read it and treat its `# budget` line and its role values as the current choices. Otherwise start from those defaults. A line whose role is not in step 5, such as `how critics`, is from a retired role. Drop it.

### 3. Budget, map, and confirm

**(a) Ask for a budget.** Prefer `ask_question` over free text. Offer these four options with these exact labels, and name the current budget when the rule records one. With no rule, say that `large` matches the skill defaults.

- `unlimited - pro reasoning`
- `large - pro and flash split`
- `medium - flash reasoning`
- `small - flash_lite reasoning`

**(b) Apply it.** Build the working table from the skill defaults, and on a re-run keep any role you changed by tier, list, or alias (`inherit`, `inherit-parent`, `auto`). `unlimited` upgrades `flash` roles to `pro`. `large` keeps the skill defaults (`pro` for judgment, prose, and hardest tasks; `flash` for fast code, explorer, investigator, and worker roles). `medium` maps code and explorer roles to `flash` and judgment roles to `pro` or `flash`. `small` maps fast code and explorer roles to `flash_lite` and judgment roles to `flash`. `inherit`, `inherit-parent`, and `auto` do not change.

**(c) Show the roles and confirm.** Show every role with its model tier. Also list each line step 2 dropped. Ask whether to accept as-is or change specific roles, offering `pro`, `flash`, `flash_lite`, and `inherit` (which runs that role on the parent chat model) as the options. Prefer `ask_question` over free text. For panel roles (arena runners, architect runners, interrogate reviewers) the value is a list, and one subagent runs per entry, alias entries included, so the list length sets the count. `arena cross-judge pool` is also a list, and Arena selects one value from it that differs from the parent's tier when possible. `swarm workers` is the default tier for every worker unless a race or comparison assigns another tier per arm.

### 4. Validate

Every tier written must be one of `pro`, `flash`, `flash_lite`, `inherit`, `inherit-parent`, or `auto`.

### 5. Write the rule

Write `~/.gemini/config/rules/pstack-models.md` as plain Markdown (no YAML frontmatter) with a `# budget` line and one line per role, using the same labels poteto-mode uses. Overwrite the whole file so re-runs stay idempotent. Shape:

```
# pstack model configuration. One line per role. Delete a line to fall back to the skill default.
# `inherit`, `inherit-parent`, or `auto` as a value: the role runs on the parent chat model (`Model: "inherit"`). Alias entries in a panel list still count toward its fan-out.
# budget: large (pro / flash)
feature, refactoring: flash
bug-fix: flash
perf-issue: flash
hillclimb: flash
judgment and prose: pro
hardest tasks: pro
how explorer: flash
how explainer: pro
why investigators: flash
why synthesizer: pro
reflect tooling: flash
reflect judgment, divergent, synthesizer: pro
arena runners: pro, flash
arena cross-judge pool: pro, flash
swarm workers: flash
architect runners: pro, flash
interrogate reviewers: pro, flash
```

### 6. Confirm

Tell the user the rule was written and that it applies to new sessions. Re-running this skill updates it.

### 7. Offer a verification skill (optional)

Check whether the project has a way to drive the real app for proof (a `verify-*` skill, or an existing harness). If not, offer once: "want a project-local verification skill, so agents can drive the app the way a user does and prove changes work? I can generate one with /create-verification-skill." On yes, invoke `/create-verification-skill` (resolves wherever pstack is installed: workspace, user, or plugin). On no, move on without pushing.
