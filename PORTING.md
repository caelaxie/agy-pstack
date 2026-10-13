# Porting `pstack` to a Target Agent Harness

This guide describes how to port `pstack` to a new agent harness while preserving compatibility with all currently supported harnesses.

## 1. Map the target harness primitives

Before editing files, identify how the target harness implements the eight host surfaces that `pstack` relies on:

| Surface | What `pstack` uses it for | What to identify in the target harness |
|---|---|---|
| **Plugin manifest** | Registers `skills/` and `agents/` with the host | Manifest file path, schema, and required fields (for example, `.<harness>-plugin/plugin.json`) |
| **Skill discovery and frontmatter** | Loads `skills/<name>/SKILL.md` and workspace skills | Frontmatter validation rules (such as whether `name` must match `<name>`), supported frontmatter keys, and `.agents/skills/` support |
| **Custom subagents** | Registers `poteto-agent` and `comment-sicko` from `agents/*.md` | Frontmatter keys for background execution and read-only permissions, plus installed plugin directories |
| **Subagent spawn and resume** | Delegates work, fans out parallel workers, and sends follow-up instructions | Tool name, custom agent type parameter, model tier parameter, worktree isolation option, and resume/message mechanism |
| **Structured user questions** | Prompts the user with selectable options during setup, triage, or catch-up | Interactive question tool name (or plain-text fallback when no tool exists) |
| **Todo and checklist state** | Tracks playbook steps verbatim from start to finish | Checklist tool or task artifact location |
| **Timers and autonomous loops** | Drives recurring wakeups for overnight runs, PR babysitting, and orchestration | Recurring timer tool, cron mechanism, or loop command |
| **Local storage and runtime tools** | Reads transcripts, persists model rules, discovers MCPs, and verifies UI/CLI behavior | Always-on workspace rule path, session transcript directory, MCP discovery path, browser automation tools, and persistent shell tools |

## 2. Add an isolated plugin manifest and validate skill frontmatter

Keep existing plugin manifests untouched so the repository continues to load in every supported harness:

1. Add the target harness manifest in its own isolated path (pointing `"skills": "./skills/"` and `"agents": "./agents/"`) rather than overwriting existing manifests.
2. Verify that every `skills/<dir>/SKILL.md` has a YAML frontmatter `name` that matches its parent directory `<dir>` (`^[a-z0-9-]+$`). Strict harness validators reject skills when `name` differs from the folder name (such as `skills/poteto-mode/SKILL.md` and `skills/make-bot-ui/SKILL.md`).
3. Confirm that optional frontmatter fields used by `pstack` (`disable-model-invocation: true`) are either supported or ignored cleanly by the target harness validator.

## 3. Standardize skill paths on `.agents/skills/`

`pstack` standardizes on `.agents/skills/` (project scope) and `~/.agents/skills/` (personal scope) for all generated and user-authored skills.

Do not introduce harness-specific skill directories in skill prompts or docs. Ensure every skill generator and reference points to `.agents/skills/` and `~/.agents/skills/`:

- `skills/create-verification-skill/SKILL.md`
- `skills/maintain-verification-skill/SKILL.md`
- `skills/automate-me/SKILL.md`
- `skills/setup-pstack/SKILL.md`
- `skills/reflect/SKILL.md`
- `skills/reflect/references/divergent-reviewer.md`
- `skills/reflect/references/judgment-reviewer.md`
- `skills/reflect/references/tooling-reviewer.md`
- `skills/reflect/references/synthesizer.md`
- `skills/poteto-help/SKILL.md`
- `docs/guide/01-setup.md`
- `docs/guide/06-verify-and-ship.md`
- `docs/guide/09-make-it-yours.md`

## 4. Update custom agent definitions in `agents/`

Update `agents/poteto-agent.md` and `agents/comment-sicko.md` so the target harness registers both custom subagents:

1. **Frontmatter keys**: Add any background-execution or permission frontmatter keys required by the target harness alongside existing keys in `agents/poteto-agent.md` and `agents/comment-sicko.md`.
2. **Installed plugin path fallback**: In `agents/poteto-agent.md` Step 1, `poteto-agent` searches for `skills/poteto-mode/SKILL.md` when running inside a consumer repository. Append the target harness's global and workspace plugin installation directories to that fallback list.

## 5. Register the harness in `skills/poteto-mode/SKILL.md`

`skills/poteto-mode/SKILL.md` is the central router for `/poteto-mode` and `poteto-agent`. Update three sections:

1. **`## Host primitives`**: Add the target harness's equivalents for subagent spawning, model selection, worktree isolation, user questions, todo tracking, recurring loops/timers, UI/CLI verification, session transcripts, and model rules.
2. **`## Non-negotiables`**: Include the target harness's subagent tool and agent type parameter so the requirement to spawn `poteto-agent` (and never a generic built-in subagent) applies on the target harness.
3. **`## Subagents`**: Document how to pass model overrides, isolate worktrees across parallel subagents, and resume or message an existing subagent on the target harness.

## 6. Thread host primitives through skills, playbooks, and docs

Update the skills, playbooks, and docs that call host tools or reference local directories directly:

1. **Model configuration (`skills/setup-pstack/SKILL.md`)**:
   - Document how the target harness lists available models or maps model tiers (`code`, `judgment`, `arena`, `interrogate`, `reflect`, `verify`).
   - Document where the target harness stores always-on workspace instructions so every skill can read the configured model table.
2. **Parallel fan-out and review skills**:
   - `skills/how/SKILL.md` and `skills/how/references/explainer-prompt.md`
   - `skills/why/SKILL.md`
   - `skills/arena/SKILL.md`
   - `skills/swarm/SKILL.md`
   - `skills/architect/SKILL.md`
   - `skills/interrogate/SKILL.md`
   - `skills/no-comments/SKILL.md`
   - `skills/reflect/SKILL.md`
   Add the target harness's subagent tool, model parameter, worktree isolation option, and MCP discovery path to each file.
3. **Transcript-reading skills**:
   - `skills/recall/SKILL.md`
   - `skills/automate-me/SKILL.md`
   - `skills/show-me-your-work/SKILL.md`
   Add the target harness's local session transcript path and format (for example, JSONL vs plain text).
4. **Interactive and visual skills**:
   - `skills/ketchup/SKILL.md`
   - `skills/teach/SKILL.md`
   - `skills/poteto-help/SKILL.md`, `skills/poteto-help/references/prompting.md`, and `skills/poteto-help/references/recipes.md`
   Add the target harness's structured question tool, image generation tool or subagent, and recurring loop command.
5. **Playbooks (`skills/poteto-mode/playbooks/*.md`)**:
   - Update host tool references in `authoring-a-skill.md`, `autonomous-run.md`, `autopilot-full.md`, `autopilot-stack.md`, `babysit.md`, `bug-fix.md`, `eval.md`, `feature.md`, `hillclimb.md`, `multi-phase-plan.md`, `opening-a-pr.md`, `orchestrate.md`, `perf-issue.md`, `refactoring.md`, `session-pickup.md`, `shipping.md`, and `visual-parity.md`.
   - When editing `skills/poteto-mode/playbooks/multi-phase-plan.md`, obey the prose rules enforced by `skills/poteto-mode/scripts/check-plan.mjs`: no em or en dashes, no curly quotes, and no mid-sentence colons (`: \S`) outside backticks.
6. **Guide and automation docs**:
   - Update `docs/guide/{01-setup,02-poteto-mode,06-verify-and-ship,07-overnight,09-make-it-yours,10-recipes-and-pitfalls}.md`.
   - Update `automations/benny/{FOR_AGENTS.md,README.md,skills/setup-benny/SKILL.md}` if the target harness supports scheduled or webhook-triggered background agents.

## 7. Update executable scripts in `skills/poteto-mode/scripts/`

Two scripts under `skills/poteto-mode/scripts/` encode host-specific checks:

1. **`skills/poteto-mode/scripts/check-plan.mjs`**:
   - Validates multi-phase plan documents and requires `## Program checklist` to reference a recurring wakeup mechanism.
   - If the target harness uses a new timer tool or loop command, add that marker to the `Program checklist` check in `skills/poteto-mode/scripts/check-plan.mjs` and add a corresponding unit test in `skills/poteto-mode/scripts/check-plan.test.ts`.
2. **`skills/poteto-mode/scripts/worktree-audit.sh`**:
   - Scans local agent transcripts modified within the last 6 hours before marking a git worktree `SAFE_TO_REMOVE`.
   - Add a transcript activity scan block for the target harness's local session log directory so `worktree-cleanup` never prunes a worktree in use by an active session.

## 8. Verify the port

Run every check below from the repository root before opening a pull request:

1. **Run plugin validation**:
   Run the target harness's plugin validator (along with any existing harness validators installed locally) and confirm all 53 skills and 2 agents pass with zero errors and zero warnings.

2. **Run the script test suite**:
   ```bash
   bun test skills/poteto-mode/scripts
   ```
   Confirm all unit tests in `check-plan.test.ts`, `watch-pr/*.test.ts`, and `orch/orch.test.ts` pass.

3. **Check shell script syntax**:
   ```bash
   bash -n skills/poteto-mode/scripts/worktree-audit.sh
   ```

4. **Check `.agents/skills/` path hygiene**:
   ```bash
   rg '\.cursor/skills'
   ```
   Confirm this returns zero matches (and verify no other harness-specific skill directories were introduced). All project and personal skill references must use `.agents/skills` or `~/.agents/skills`.

5. **Check house prose rules on changed lines**:
   ```bash
   git diff | rg '^[+][^+]..*[\u2013\u2014]'
   ```
   Confirm zero added lines contain en dashes (`–`) or em dashes (`—`).
