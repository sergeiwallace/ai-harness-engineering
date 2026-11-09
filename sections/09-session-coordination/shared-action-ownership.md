---
title: "Owning shared actions"
parent: "Coordinator and chief-of-staff sessions"
grand_parent: "Coordinating several sessions"
nav_order: 1
---

{% include page_nav.html %}

# Owning shared actions

As of 2026-11-09.

A sub-topic of [coordinator and chief-of-staff sessions]({{ '/sections/09-session-coordination/coordinator-sessions.html' | relative_url }}). It covers which actions several sessions must not run at once, how one owner per machine turns races into a queue, and how to keep that owner from becoming the bottleneck.

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

Running the tests on `main` after each merge catches this at the merge that caused it. Merge queues on hosted Git services use the same idea, covered in [code review]({{ '/sections/13-code-review/' | relative_url }}).

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

An owner session can exit, crash, or be compacting. Workers need a rule for that case that is not "do it yourself":

- Wait and retry for a bounded time.
- Tell the top coordinator, which can start a new owner.
- Never take the action in parallel "just this once". That is how the race comes back.

## Examples from my own harness

- One chief-of-staff session per machine runs merges, pushes to `main` and installs; workers send it "ready to merge".
- A machine-wide test lock, so parallel test runs from several sessions do not overload the machine.
- A machine contention check that sessions read before starting a sub-agent or a parallel test run.

To start: decide which one session may merge to `main`, and tell the others in their context file.

## Sources

- `flock(1)`: <https://man7.org/linux/man-pages/man1/flock.1.html>.

{% include page_nav.html %}
