---
title: "Owning shared actions"
parent: "Coordinator and chief-of-staff sessions"
grand_parent: "Coordinating several sessions"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Owning shared actions

A sub-topic of [coordinator and chief-of-staff sessions]({{ '/sections/09-session-coordination/coordinator-sessions.html' | relative_url }}). It covers which actions several sessions must not run at once, how one owner per machine turns races into a queue, how to enforce the ownership so a worker that forgets is refused, locks for actions that need no owner, and how to keep the owner from becoming the bottleneck.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Safe and unsafe in parallel

Sessions that each work in their own [worktree]({{ '/sections/08-delegation/' | relative_url }}) can edit, build and test in parallel without touching each other. Some actions reach outside the worktree, into something every session shares.

| Action | Shared thing | What goes wrong in parallel |
|---|---|---|
| Merge to `main` | The main branch | Two merges each pass their tests alone and break together |
| Push to a shared branch | The remote branch | One push is rejected; the session retries with a rebase it did not review |
| Install or update a shared tool | The machine's tool directory | A half-written install while another session runs the tool |
| Heavy test run | CPU and memory | The machine slows down and every session's tests time out |
| Sync the main checkout | The main working tree | One sync moves files under another session's feet |

One way to handle these is to give each shared action one owner session per machine, which the other sessions ask. A lock, below, is another.

## The owner pattern

![A sequence diagram. Worker A and worker B both ask the chief to merge. Inside a critical fragment the chief merges A into main, runs the tests, gets green, and confirms to A; then merges B, runs the tests again, gets green and confirms to B.]({{ '/assets/images/09-session-coordination/merge-owner.svg' | relative_url }}){: style="width:100%"}

*Figure: two merge requests become a queue under one owner. Made with archify.*

A worker that finishes a change does not merge it. It sends the owner a short request: the branch, what it changes, and the evidence that it passed. The owner merges requests one at a time, runs the affected tests on the merged result, and replies.

### Example: a merge request message

```text
ready to merge
branch: wt/fix-export-index
change: restore the index the export query relies on
evidence: test_export_uses_index passes; full suite green at 4e1a9c2
```

### Why test the merged result

Two branches can each pass alone and fail together.

1. Branch A renames `fetch_rows` to `load_rows` and updates every caller it can see.
2. Branch B adds a new caller of `fetch_rows`.
3. Each branch's tests pass on its own.
4. After both merge, `main` fails: B's new caller uses a name that no longer exists.

<!-- IMAGE NEEDED: two git branches that each pass tests and break main when both merge; search: git branch merge diagram, semantic merge conflict, merge queue diagram -->

Running the tests on `main` after each merge catches this at the merge that caused it. Merge queues on hosted Git services use the same idea, covered in [code review]({{ '/sections/13-code-review/' | relative_url }}).

## Enforcing ownership

A line in each worker's context file saying "do not merge" is the first layer, and the one a busy model skips. The layers that enforce it sit in the tools:

| Layer | What it does here | Enforces? |
|---|---|---|
| 1. Instructions | Each worker's context file says the chief merges | Guides |
| 2. Workflow | The worker's "finish" skill ends by sending "ready to merge", with no merge step in it | Yes, for runs that use the skill |
| 3. Hook | A `PreToolUse` hook refuses `git push` to `main` and `git merge` into `main` in any session that is not the owner | Yes, at the moment of action |
| 4. Server side | Branch protection on `main`, or a merge queue, so even a push from outside a session cannot land unchecked | Yes, for every client |

### Example: a hook that refuses non-owner merges

A hook script receives the tool call as JSON on standard input. Exit code 2 blocks the call and passes the message on standard error back to Claude.

```bash
#!/usr/bin/env bash
# PreToolUse hook for Bash: only the owner session may push or merge to main.
set -euo pipefail
cmd="$(jq -r '.tool_input.command // empty')"
if [[ "${SESSION_ROLE:-worker}" != "owner" ]] &&
   [[ "$cmd" =~ git[[:space:]]+(push[^|;&]*[[:space:]]main|merge) ]]; then
  echo "Only the owner session merges or pushes to main. Send the chief 'ready to merge' with your branch." >&2
  exit 2
fi
exit 0
```

The owner session is started with `SESSION_ROLE=owner` in its environment. The pattern is deliberately rough: it errs toward refusing, and the message tells the worker what to do instead.

## Locks for things that are not merges

Some shared resources are short and frequent to use, and routing each use through a session costs more than it saves. A machine-wide lock covers them.

| Resource | Lock or owner? | Why |
|---|---|---|
| Merges to `main` | Owner | Needs tests on the merged result |
| Heavy test runs | Lock | Short, frequent, no judgment needed |
| Shared tool installs | Owner | Needs a check that nothing is running the tool |

### Example: a test lock

```bash
# take the machine's test lock, run, release; wait up to 30 minutes
flock --timeout 1800 /var/lock/heavy-tests.lock \
  pytest -n 6 tests/
```

`flock` releases the lock when the command exits, even if it crashes. A lock that a crashed holder cannot release blocks every other session until someone notices.

## Keeping the owner from becoming a bottleneck

A queue moves work out of a race and into a line. If the line gets long, the owner is now the slowest part.

- Merge only what is ready. A request without evidence goes back to the worker, so the owner never debugs.
- Run the tests the change affects on each merge, and the full suite on a schedule. Running the full suite after every merge is the usual cause of a slow queue; [testing]({{ '/sections/12-testing/' | relative_url }}) covers test selection.
- Measure where the time goes. If most of the wait is test time, speed up tests; if it is the owner being busy with other work, narrow the owner's job.

### Example: a slow queue

1. Five workers finish within ten minutes.
2. First setup: the chief runs the full twenty-minute suite after every merge.
   - The last change lands after nearly two hours.
3. Second setup: the chief runs only the affected tests per merge, plus one full run at the end.
   - The same five land in about thirty minutes.
   - The full run at the end still catches anything the affected tests missed.

## When the owner is not there

An owner session can exit, crash, or be compacting. Workers need a rule for that case other than doing the action themselves:

1. Wait and retry for a bounded time, for example three tries over fifteen minutes.
2. Tell the top coordinator, which can start a new owner.
3. Keep the change on its branch until an owner takes it.

The hook above makes "just this once" unavailable to a worker, so the race cannot come back by accident.

## Examples from my own harness

- One chief-of-staff session per machine runs merges, pushes to `main` and installs; workers send it "ready to merge".
- A machine-wide test lock, so parallel test runs from several sessions do not overload the machine.
- A machine contention check that sessions read before starting a sub-agent or a parallel test run.

To start: decide which one session may merge to `main`, enforce it with branch protection or a hook like the one above, and add the context-file line as the second layer.

## Sources

- Hooks, `PreToolUse` and exit code 2: [Hooks reference (Claude Code docs)](https://code.claude.com/docs/en/hooks.md).
- `flock(1)`: [flock(1) manual page](https://man7.org/linux/man-pages/man1/flock.1.html).

{% include page_nav.html %}
