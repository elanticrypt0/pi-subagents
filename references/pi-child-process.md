# Running a child `pi` process as a subagent

Reference for `pi-subagents/SKILL.md`, for sessions where the native subagent tool
family is not available. Flags and behaviors verified against `pi --help` and live
runs on Pi 1.3.0.

## Shape

```sh
cd <target-worktree> && pi -p --no-session --model provider/id --thinking medium \
  --tools read,grep,find,ls "<self-contained task>" > out.txt 2> err.txt
```

- `cd` inside the command. cwd decides project config, `AGENTS.md`, skill discovery, `@path` resolution, and session grouping; nothing else reaches the child.
- `-p` / `--print` is mandatory for a delegated child. Without it, a run whose stdin and stdout are terminals opens the TUI and waits for a human.
- Default output is text: only the final assistant text reaches stdout. `--mode json` writes a JSONL event stream instead — a `session` header, then `agent_end` for each low-level run and `agent_settled` when nothing automatic remains — and reserves stdout for protocol records; diagnostics go to stderr. Never mix `--mode json` with human-facing output.
- A child has no terminal for consent. Give it every decision up front, and grant only tools that cannot stop mid-task waiting for an answer.

## Flags that decide the child's capability

| Flag | Effect |
| --- | --- |
| `--tools read,grep,find,ls` | Allowlist replacing the default selection: read-only explorer |
| `--tools read,grep,edit,write` | Bounded writer; add `bash` explicitly when it must run tests or builds |
| `--exclude-tools <list>` | Denylist applied on top of the selection; accepts `*` patterns |
| `--no-tools` / `--no-builtin-tools` | No tools at all / builtin tools off but extension tools stay |
| `--model provider/id` | Pins the model; an unknown id exits 1 with `Model "…" not found`, so check `pi --list-models` first |
| `--thinking <level>` | `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`; clamped to the model |
| `--no-session` | Ephemeral: nothing lands in the session store |
| `--session-id <id>` | Deterministic session id; created if absent, resumed when reused |
| `-n <name>` | Session display name, so the child is recognizable later |
| `--append-system-prompt <text\|file>` | Narrow the role while keeping Pi's system prompt |
| `--no-extensions` | Strip extensions and the primary-session prompt: a plain executor with no orchestration of its own |
| `--skill <path>` / `--no-skills` | Add or suppress skills |
| `--no-context-files` | Skip `AGENTS.md` / `CLAUDE.md` discovery |
| `--approve` | Trust project-local files for this run |

A `pi` child loads the same global extensions and primary-session prompt as the
parent, so by default it can behave like a second orchestrator. Narrow the role
with `--append-system-prompt`, or use `--no-extensions` when the child must be a
plain executor that only reads, edits, and reports.

## Recipes

Read-only explorer:

```sh
cd "$WT" && pi -p --no-session --tools read,grep,find,ls --model "$M" \
  "Map <area>. Reply with at most 40 lines of path:line evidence. Do not edit anything."
```

Bounded writer, surfaces declared in the text the child receives:

```sh
cd "$WT" && pi -p --no-session --tools read,grep,edit,write,bash --model "$M" \
  "## Allowed edit surfaces
src/parser.ts
test/parser.test.ts

Implement S3 ... Run node --test test/parser.test.ts and report the exact result."
```

Verifier for a named risk:

```sh
cd "$WT" && pi -p --session-id verify-T2 --tools read,grep,bash --model "$M" \
  "Risk: item 3 (shared parser). Re-run node --test and report each assertion, pass or fail."
```

Background capture, parent side: launch the command under the background bash
policy with stdout and stderr redirected to files, treat the exit notice as the
completion event, and read the stderr tail first — a child that failed to start
writes only stderr.

## Parallel fan-out with worktrees

Two writers in one worktree fight over the same index and the same files. When the
parent has 2+ independent units, give each child its own worktree; this is the
process-child equivalent of `subagent_run` with `workspace_root`.

```sh
ROOT="$PWD"; OUT="$(mktemp -d)"
for T in T2 T3; do
  git worktree add "$(dirname "$ROOT")/wt-$T" -b "work/$T" HEAD
done
```

Then launch one background job per child, outputs kept **outside** every worktree
so they never show up as untracked files in a review diff:

```sh
cd "$(dirname "$ROOT")/wt-T2" && pi -p --session-id fanout-T2 \
  --tools read,grep,edit,write,bash --model "$M" "$TASK_T2" \
  > "$OUT/T2.out" 2> "$OUT/T2.err"
```

Rules that matter here:

- **Disjointness is the parent's job.** No runtime sees two separate `pi` processes, so no admission check rejects overlapping surfaces. Compute disjointness yourself: overlap now surfaces later as a merge conflict, not as a launch error.
- **Keep the session id** in fan-out (`--session-id fanout-T2`) instead of `--no-session`: a failed child is then resumable with a follow-up instead of restarted from zero.
- **A worktree starts at the commit you name**, so the parent's uncommitted work is not there. Either commit first, or ship it as a patch: `git -C "$ROOT" diff > "$OUT/wip.patch"` and `git -C <worktree> apply "$OUT/wip.patch"`. Keep `$OUT` absolute (`mktemp -d` with no template) and it survives `git -C` changing the working directory; a relative patch path fails with `no such file or directory`.
- **Wait for all exit notices before merging.** Never open a review of one worktree while a sibling still writes to its own; collect every job, then read each `git -C <worktree> diff`.
- **Children still do not commit or push.** If one does commit its unit, the parent merges (`git merge work/T2`) or cherry-picks; otherwise the parent applies the diff. The parent alone decides what enters the feature branch.
- **`git worktree add` refuses an existing branch.** Use a fresh branch name, or `-B` to reset one you own.
- **Cleanup is destructive.** `git worktree remove <path>` refuses a dirty worktree; `--force` discards the child's work, so confirm with the human before using it. After removal, `git branch -D work/T2` drops a branch you decided to reject.

## Resume and continuation

```sh
pi -p --session-id <id> --tools read,grep --model "$M" "<follow-up>"
```

Reusing the id continues the same conversation, so the follow-up sees the child's
own prior turns and still none of the parent's context. `--continue` picks the
most recent session for the current project, which is nondeterministic once
children run in parallel; always name the id. A process child has no steering and
no `subagent_continue` equivalent: relaunch with a corrected task, or reuse the
session id to keep its context.

## Failure signatures

| Symptom | Cause |
| --- | --- |
| Command hangs with no output | Missing `-p`, so the TUI opened |
| Exit 1, `Model "…" not found` | Wrong `--model` id or an unavailable provider |
| Answer ignores the repository | Wrong cwd, or `@path` resolved against a different directory |
| Child asks a question and stops | No terminal for consent; put the decision in the task |
| JSONL parsing errors | Human output mixed into `--mode json` stdout, or records split on Unicode separators instead of LF |
| Child starts orchestrating instead of executing | Inherited primary-session prompt; narrow with `--append-system-prompt` or `--no-extensions` |
