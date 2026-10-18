---
name: pi-subagents
description: "Trigger: subagente, subagent, delegar a un agente, define an agent, background subagent, subagent_run, pi -p. Launch Pi subagents: the native subagent tools when the session exposes them, a child `pi` process otherwise."
license: Apache-2.0
metadata:
  author: elanticrypt0
  version: "1.1"
---

# Pi Subagents

## Activation Contract

Load when work is delegated to a subagent, when a subagent definition must be created or fixed, or when the parent launches a child `pi` process: launch, steer, resume, cancel, or collect a result.

## What a subagent is

A subagent is a second Pi instance, not a tool call inside the parent's process.

- **Own session and context window.** None of the parent's messages, todo list, or scratch state transfers. The child knows only what it reads from disk and what the task says.
- **Own tool inventory and permissions.** The parent's tools do not follow the child, and the child's report carries no authority the parent's own tools would grant.
- **Own working directory.** cwd decides project config, `AGENTS.md`, skill discovery, `@path` resolution, and where the session is stored.
- **Text plus an exit status.** The parent alone decides what that output means and what happens next.

Two launchers, one contract: the native subagent tool family when the session exposes it (`subagent_list_agents` answers), a `pi` child process through bash otherwise. Pick the launcher, then apply the same handoff rules.

## Hard Rules

- Discover before launching: `subagent_list_agents` when the tool exists, `pi --list-models` when it does not. Never guess an agent name, and never assume its declared `tools` cover the task: a read-only explorer cannot write, a reviewer carries no review authority.
- A child starts with none of the parent's context. Give it a self-contained `task` plus optional `context`, and hand off by reference (`Spec: odd/tasks/<feature>.md, T2, S3`) instead of paraphrasing requirements the child must meet.
- Every writer launch carries an exact `## Allowed edit surfaces` heading with narrow repository-relative paths or globs, one per line, never `.` or an absolute path. A runtime that recognizes its own writer agents enforces it and rejects overlapping parallel writers in one worktree at queue time; the heading is still the only scope declaration parent and child share, so write it even when nothing validates it.
- Never poll, sleep, or re-launch a background child. Native: end the turn and resume on the result message. Process: end the turn and resume on the background job's exit notice.
- Parallel writers only with disjoint edit surfaces, or in isolated worktrees (`workspace_root` for a native child, `git worktree add` for a process child).
- Children do not commit, push, publish, or accept user decisions on the parent's behalf.

## Decision Gates

| Need | Action |
| --- | --- |
| Map an unknown area, context-heavy reading | read-only explorer: an agent with no write tools, or `--tools read,grep,find,ls`; request a handoff of at most ~2k tokens with `path:line` evidence |
| One bounded writer unit | writer agent, or a `pi` child with write tools: spec reference plus edit surfaces |
| Independent check of a named high-risk change | verifier agent, or a `pi` child whose task names the `Risk: item N` line |
| Single small edit the parent can do | inline work; a child spends one to two minutes reading before it writes |
| Agent file format, discovery, model routing | `references/agent-frontmatter.md` |
| Exact `pi` child flags, recipes, failure signatures | `references/pi-child-process.md` |

Mode: `background` when the parent can keep working without the answer; `task` when the next step needs it. A process child is always asynchronous to the parent: launch it under the background bash policy and treat its exit as the result event. Native background launches are rejected in the single-shot hosts `pi -p` and `pi --mode json`, which exit before a result could arrive.

## Execution Steps

1. Discover: `subagent_list_agents` → pick the agent whose declared tools fit the task. No tool family, or no agent that fits → build a `pi` child instead of weakening the handoff.
2. Build the task: objective, spec reference, acceptance criteria, edit surfaces for writers, and the checks the child must run.
3. Launch:
   - Native: `subagent_run` with `agent`, `task`, `context`, and a 3–6 word `label`. Foreground consumes the result now; background ends the turn.
   - Process: `cd <worktree> && pi -p …` under the background bash policy, stdout and stderr captured to files.
4. While it runs: `subagent_send_message` to steer, `subagent_cancel` to stop; a child that asks a question gets exactly one `subagent_reply`. A process child cannot be steered mid-run: stop the job or let it finish, then relaunch with a corrected task.
5. Resume finished work: `subagent_continue` for a native task; `pi -p --session-id <id>` for a process child you gave an id.
6. Inspect the returned evidence and re-run the deciding check in the parent before reporting the outcome.

## Output Contract

Report: launcher and mode used, task id or child session id, scope actually touched, checks run with exact results, and any blocker the child surfaced. Name the fallback whenever the native tool family was unavailable.

## References

- `references/agent-frontmatter.md` — agent file shape, frontmatter keys, discovery precedence, `subagents.json` defaults, tool families, failure modes.
- `references/pi-child-process.md` — `pi` child invocation: flags that decide capability, explorer and writer recipes, background capture, parallel fan-out in worktrees, resume, failure signatures.
