---
name: reflect
description: Spawn three parallel review subagents over the active transcript, surface learnings, and route each to a concrete edit on an existing skill. Use when the user says reflect.
disable-model-invocation: true
---

# Reflect

Mine the current conversation for durable learnings, then route them into skill edits.

Invoke when the user says "reflect" or "/reflect". Skip when the conversation is trivial, off-topic, or already covered by an existing skill the parent followed correctly. One-offs are not learnings.

## 1. Locate the active transcript

Before fanning out, find this conversation's transcript at `<appDataDir>/brain/<conversation-id>/.system_generated/logs/transcript.jsonl` (the system prompt names `<appDataDir>` and `<conversation-id>`). Do not glob across unrelated workspaces.

If the transcript file is unavailable, pass a tight digest of the session instead.

## 2. Spawn three reviewers in parallel

One `invoke_subagent` call with three entries, `TypeName: "self"` (agent mode). Reviewers need MCP access to look up the tickets, chat threads, and observability traces the transcript references, and `TypeName: "research"` strips MCPs.

Each reviewer and the synthesizer name a role line in `~/.gemini/config/rules/pstack-models.md` and a default. That file is an always-applied rule, already in context when present, so do not `view_file` it. Set `Model` to that line's value, or to the default if the rule is not in context or the line is missing. Pass `Model: "inherit"` when the value is `inherit`, `auto`, or `inherit-parent`.

| Lens | Role line | Default `Model` | Prompt template |
|---|---|---|---|
| Judgment | `reflect judgment, divergent, synthesizer` | `pro` | `references/judgment-reviewer.md` |
| Tooling | `reflect tooling` | `flash` | `references/tooling-reviewer.md` |
| Divergent | `reflect judgment, divergent, synthesizer` | `pro` | `references/divergent-reviewer.md` |

Pass each template verbatim, substituting the transcript path or digest where marked. Reviewers return findings in their subagent response body.

## 3. Synthesize

One `invoke_subagent` call, `TypeName: "self"`, `Model` from the `reflect judgment, divergent, synthesizer` line (default `pro`). It spot-verifies citations through MCP. Pass `references/synthesizer.md` verbatim, with each reviewer's full output inlined where marked. It returns an Accepted / Rejected / Backlog list.

## 4. Structural enforcement check

Move any Accepted item that a lint rule, script, metadata flag, or runtime check would enforce more reliably to Backlog. See the **encode-lessons-in-structure** principle skill.

## 5. Apply

Present the synthesizer's full Accepted / Rejected / Backlog output and wait for explicit approval before applying any Accepted edit. The user picks the subset and may redirect routings. Skill changes affect every future agent in the org. Do not auto-apply.

File each Backlog item to your team's devex or backlog tracker without waiting. Only the Accepted list waits for approval.

Follow each approved row's Routing exactly:

- Trivial existing-skill edit (a one-line bullet, a tightened sentence, a stale fact corrected): the parent does it directly.
- Substantive existing-skill edit (a new section, a new pattern table, more than ~10 lines): follow Antigravity's built-in `agy-customizations` skill.
- `tune description: <skill path>` (the skill exists but didn't trigger when it should have): tune its frontmatter description per `agy-customizations`.
- `new skill via agy-customizations: <kebab-name>`: create the skill following `agy-customizations`. Do not invent the shape ad hoc.

Run `agy plugin validate` on touched plugin skills before declaring done.

## 6. Summarize for the user

Short list, no preamble:

- Edits applied: `<skill path>`. What changed, one line each.
- New skills created: `<skill path>`. One line each (rare).
- Backlog filed to the devex tracker: `<issue title>` (`<tags>`). One line each.
- Dropped: one line per rejected finding + reason from the synthesizer.
