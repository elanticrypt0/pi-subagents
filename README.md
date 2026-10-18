# pi-subagents

An Agent Skill that teaches a Pi session how to delegate work to a **subagent** —
understood literally as a second Pi instance, not as a tool call in the parent's
process.

It covers both launchers with one contract:

- the native subagent tool family (`subagent_run`, `subagent_status`,
  `subagent_continue`, …) when the running session exposes it;
- a child `pi` process launched through bash (`pi -p`, `--mode json`) when it does
  not.

Nothing here depends on a particular vendor pack of agent names, so the same rules
apply to an arbitrary agent definition or to a bare `pi` child.

## When a model loads it

The frontmatter description routes on `subagent`, `subagent_run`, `pi -p`,
"delegar a un agente", and "define an agent". Pi advertises only the name,
description, and path at startup; the body is read when the task matches, or on
demand with `/skill:pi-subagents`.

## Layout

```text
pi-subagents/
├── SKILL.md                          # entry point: model, rules, gates, steps
├── references/
│   ├── agent-frontmatter.md          # agent file shape, discovery, runtime config
│   └── pi-child-process.md           # `pi` child flags, recipes, fan-out, failures
└── .atl/                             # generated registry metadata, not hand-edited
```

`SKILL.md` is the source of truth. The reference pages hold detail that would only
consume context if it were inlined, and both are linked from the skill body.

## What it establishes

- **The mental model.** A child has its own session, context window, tool
  inventory, and working directory; it returns text plus an exit status, and the
  parent decides what that means.
- **The handoff.** A self-contained task plus optional context, handed off by
  reference (`Spec: odd/tasks/<feature>.md, T2, S3`), never by paraphrase.
- **The writer contract.** Every writer launch carries an exact
  `## Allowed edit surfaces` heading. A runtime enforces it only for the writer
  agents it recognizes, which is why the document tells the parent to write it
  regardless.
- **Waiting discipline.** No polling and no relaunching: end the turn and resume on
  the native result message or on the background job's exit notice.
- **Isolation.** Parallel writers need disjoint surfaces or separate worktrees.

## Install

Copy this directory into a skill location Pi scans (directories containing a
`SKILL.md` are discovered recursively):

```sh
# user scope
cp -r pi-subagents ~/.agents/skills/pi-subagents

# project scope
cp -r pi-subagents .agents/skills/pi-subagents
```

A symlink works only if your setup follows it during discovery; copying is the
verified form.

For a single child invocation, pass the skill explicitly instead:

```sh
pi -p --skill /path/to/pi-subagents/SKILL.md "…"
```

`--skill` takes a file or a directory and can be repeated; `--no-skills` disables
discovery entirely when a child must run without any skill text in its prompt.

## Verify the claims before editing

The reference pages state behaviors, so check them against the installed binary
rather than trusting this repository:

```sh
pi --help                          # flags and modes
pi --list-models                   # valid --model ids
pi -p --no-session --tools read --model <id> "Reply with only: OK"
```

A missing `-p` opens the TUI and hangs; an unknown `--model` exits 1 with
`Model "…" not found`. Both signatures, and the fan-out mechanics
(`git worktree add`, dirty-worktree removal, patch paths under `git -C`), were
confirmed by running them.

## Maintenance rules

- Keep `SKILL.md` short enough to read in one pass; new detail belongs in
  `references/`.
- Name mechanisms, not vendors: describe a capability ("writer agent", "verifier
  agent") rather than one pack's agent names.
- Do not restate `pi --help` exhaustively; document the flags that change a child's
  capability, and the failure each one produces when misused.
- Re-verify after a Pi upgrade, and update the version in the frontmatter when the
  advice changes.

## License

Apache-2.0.
