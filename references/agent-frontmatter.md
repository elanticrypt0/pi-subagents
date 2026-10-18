# Subagent definitions, discovery, and configuration

Reference for `pi-subagents/SKILL.md`. Everything here was verified against the
installed subagent runtime and the Pi skill/subagent tooling, not inferred from
prompts. Paths such as `~/.pi/agent/agents` are the runtime's own defaults; when a
runtime names its files differently, trust `subagent_list_agents` over this page.

## Agent file shape

One Markdown file per agent: YAML frontmatter plus the instruction body. The body
is the child's system-level instruction, so it is written in the imperative and
scoped to one role.

```markdown
---
name: docs-writer
description: Writes and updates repository documentation for one bounded area.
model: anthropic/claude-sonnet-4
thinking: medium
mode: task
tools:
  - read
  - grep
  - write
  - edit
---

You update documentation only. Read the target files first, keep the surrounding
style, and never change source code.
```

## Frontmatter keys

| Key | Meaning |
| --- | --- |
| `name` | Agent identity used by `subagent_run`; defaults to the file name |
| `description` | Shown by `subagent_list_agents`; this is what the parent routes on |
| `model` | `provider/id` (or a bare id resolved against the default provider) |
| `thinking` | Reasoning effort; aliases `effort` and `thinking_level` are accepted |
| `mode` / `subagent_mode` | Default launch mode: `task` or `background` |
| `tools` | List, comma list, or `- item` block of the tools the child may use |

Hard failures: a body with no instructions after the frontmatter
(`no instructions after the frontmatter`), and an unknown `thinking` or `mode`
value. The file is skipped with that error, so `subagent_list_agents` will not
show the agent at all.

`tools` is declared, not enforced: the runtime does not stop a child from calling
something else, and `subagent_list_agents` labels the list as unverified. Treat it
as the honest capability declaration a parent routes on, not as a sandbox.

## Discovery and precedence

Directories are scanned in this order, and a later definition replaces an earlier
one with the same `name`:

1. `~/.pi/agent/agents` — global
2. `~/.pi/agent/subagents` — global
3. `<repo>/.pi/agents` — project
4. `<repo>/.pi/subagents` — project

So a project agent shadows a global one of the same name, and `subagents/` shadows
`agents/` inside the same scope. Duplicate names never merge: the last one wins
silently, which is the usual cause of "the agent I edited is not the one that ran".
Deleting a project override is the fix.

## Runtime configuration

Two `subagents.json` files, merged field by field with the project value winning:
`~/.pi/agent/subagents.json` (global) and `<repo>/.pi/subagents.json` (project).

| Key | Default | Effect |
| --- | --- | --- |
| `default_model` | unset (session model) | Model for agents that name none |
| `default_effort` (aliases `default_thinking_level`, `default_thinking`) | unset | Reasoning effort fallback |
| `default_mode` | `task` | Mode for a launch that names none |
| `model_profiles` | `{}` | Per-agent `{ "model": ..., "effort": ... }`; merges per agent, so a project can change one agent's effort without repeating its model |
| `stall_timeout_ms` | `240000` | Silence budget for a child that stops producing output |
| `tool_stall_timeout_ms` | `1800000` | Longer ceiling for a tool call that is in flight; never below the stall budget |
| `max_concurrency` | `5` | Children running at once |
| `history_max_tasks` | `200` | Retained task history |

Profile resolution per agent: `model_profiles[name]` → the agent's own `model` →
`default_model`. `timeout_ms` is accepted but inert (legacy key).

## The tool family

| Tool | Use |
| --- | --- |
| `subagent_list_agents` | Discover definitions, scope, description, declared tools |
| `subagent_run` | Launch; `mode` task waits, background returns a task id |
| `subagent_status` / `subagent_result` | Inspect one task; a foreground launch already returned its result |
| `subagent_list_tasks` | This session's tasks, newest first |
| `subagent_send_message` | Steer a running child before its next model call |
| `subagent_continue` | Resume a finished child in its own session with a new prompt |
| `subagent_cancel` | Stop a queued or running child |
| `subagent_reply` | Answer one live question from a child of this parent |

`subagent_run` targeting: `workspace_root` is a worktree in the parent's own clone;
`repository_root` is an independent repository and asks for consent first.

## Runtime-enforced delegation contracts

These are admission checks a runtime may perform when a launch is queued in the
primary session. A rejection names the exact repair, and retrying the same shape
fails the same way.

- **Writers.** The task must contain an exact `## Allowed edit surfaces` heading,
  with narrow repository-relative paths or globs, one per line, inside whole-entry
  backticks when a path contains whitespace. `.` and absolute paths are refused.
  Overlap detection is per Pi process, per canonical worktree root: an entry covers
  everything under it, a whole-segment `**` covers any depth, and `**` inside a
  segment matches within that segment only. It is a queue-time conflict check, not
  a write-time guard.
- **Scope of the checks.** Admission is keyed on the agent name, so a runtime
  enforces the writer surface rules only for the writer agents it knows. An agent
  it does not know is usually launched unvalidated, which is a reason to write the
  heading anyway, never a reason to omit it.
- **Extra requirements for particular agents.** Some agents demand an additional
  canonical line in the task, such as a spec reference naming a task, or a
  `Risk: item N` line for a verifier that must not audit its own change. Read the
  rejection text: it states the required section verbatim. Do not generalize one
  agent's rule to every child.
- **Background.** Rejected in `pi -p` and `pi --mode json`, because the host exits
  before a later result could be delivered.

## Child environment

Each child launched through the native tool family gets its own `TMPDIR` (plus
`TMP`/`TEMP` on Windows) as a fresh directory under the inherited temp root,
removed when the child ends. A continuation is a new child with a new directory.
Children never inherit the parent's context window, todo list, or session scratch.

A `pi` child process launched from bash inherits the parent shell environment
instead, including `TMPDIR`; give it an explicit `--session-dir` or `--no-session`
when several children must not share session storage. See
`references/pi-child-process.md`.

## Common failure modes

| Symptom | Cause |
| --- | --- |
| Agent missing from `subagent_list_agents` | Frontmatter parse error, empty body, or an invalid `thinking`/`mode` value |
| Edited definition does not take effect | A later discovery directory defines the same `name` |
| Launch rejected for missing surfaces | Writer without the exact `## Allowed edit surfaces` heading |
| Launch rejected for overlap | Two parallel writers claiming the same path in one worktree |
| Result never arrives | Background launch from a single-shot host, or the parent ended without waiting for the completion message |
| Child reports nothing it could not read | Handoff assumed inherited context; the task was not self-contained |
