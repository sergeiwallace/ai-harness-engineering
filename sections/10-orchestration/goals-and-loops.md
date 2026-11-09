---
title: "Goals and loops"
parent: "Orchestration and workflows"
nav_order: 3
has_children: true
has_toc: false
redirect_from:
  - /sections/10-orchestration/detail.html
---

{% include page_nav.html %}

# Goals and loops

As of 2026-11-09.

A goal keeps an agent working until a condition is met. A loop repeats a prompt on a schedule. Both turn a single answer into sustained work, and both need an exit condition and a limit. This page covers `/goal` and its evaluator, writing a condition it can judge, Stop hooks as the enforced version of a goal, `/loop` and scheduled tasks, and how each one ends. The [section overview]({{ '/sections/10-orchestration/' | relative_url }}) is the short version; a sub-topic page covers [hill-climbing and quality ratchets]({{ '/sections/10-orchestration/hill-climbing.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Three ways to keep a session running

| Approach | Next turn starts when | Stops when | Lives in |
|---|---|---|---|
| `/goal` | The previous turn finishes | A model judges the condition met or impossible, a turn fails on an error you must fix, or you clear it | The current session only |
| `/loop` | An interval elapses, or one Claude picks | You stop it, Claude ends a self-paced loop, or 7 days pass | The current session; restored on resume |
| Stop hook | The previous turn finishes | Your script or prompt allows the stop | A settings file; every session in its scope |

Auto mode is a different thing: it approves tool calls inside a turn but never starts a new turn. It pairs with `/goal` for unattended runs.

## `/goal`

![A sequence diagram of one goal. You set the goal "suite shows 0 failures". Inside a loop fragment the agent runs the test suite, gets 2 failures printed, ends its turn, and the evaluator, which only reads, answers "not met: continue". The agent fixes and runs again, gets 0 failures printed, ends its turn, and the evaluator answers "condition met". The agent reports done with the output.]({{ '/assets/images/10-orchestration/goal-run.svg' | relative_url }}){: style="width:100%"}

*Figure: a `/goal` run; the evaluator judges only what was printed. Made with archify.*

### How it works

1. You set the goal: `/goal <condition>`. Setting it starts a turn at once, with the condition as the instruction.
2. The agent works a turn and ends it.
3. Claude Code sends the condition and the conversation to the configured small fast model.
   - It calls no tools and reads no files.
   - It returns one of three verdicts, each with a short reason.
4. The verdict decides what happens next.

| Verdict | What happens |
|---|---|
| Not yet met | The agent starts another turn, with the reason as guidance |
| Met | The goal clears and an achieved entry is recorded |
| Impossible | The goal clears and a failed entry is recorded with the reason |

Under the hood, `/goal` is a session-scoped prompt-based Stop hook.

### Managing a goal

| Command | Does |
|---|---|
| `/goal <condition>` | Sets a goal (one per session; a new one replaces the old) |
| `/goal` | Shows the condition, running time, turns evaluated, token spend and the evaluator's last reason |
| `/goal clear` | Removes it; `stop`, `off`, `reset`, `none` and `cancel` also work, and `/clear` removes it too |
| `claude -p "/goal <condition>"` | Runs the whole loop in one non-interactive call; add `--output-format stream-json --verbose` to watch it |

A goal is restored when you resume the session, with its turn count, timer and token baseline reset.

### Settings

| Setting | Effect |
|---|---|
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Picks the evaluator model; it also changes every other use of the small fast model |
| `CLAUDE_CODE_GOAL_CHECKIN_MINUTES` | First check-in interval while background work keeps a goal waiting (30 by default); `0` turns off check-ins and automatic retries |

### What the evaluator can and cannot see

| Can see | Cannot see |
|---|---|
| Everything printed in the conversation | Files on disk |
| Command output the agent showed | Whether a command was really run, if its output was not shown |
| The agent's own claims | Whether those claims are true |

So the condition has to name evidence the agent will print. "The bug is fixed" invites the agent to say so. "The regression test's output, shown after the last change, reports it passing" asks for something the evaluator can read.

## Writing a condition

A condition the evaluator can judge has these properties:

| Property | Question it answers | Missing it looks like |
|---|---|---|
| One measurable end state | What does done look like? | "Improve the docs" |
| A stated check | What printed output proves it? | "The bug is fixed" |
| Constraints | What must not change on the way? | Tests "fixed" by editing them |
| A bound | When should it stop trying? | It runs until it stops making progress |
| A way out | What if it cannot finish? | It spins to the bound |

A condition can be up to 4,000 characters. To bound a goal, put a turn or time clause in the condition, such as "or stop after 20 turns"; Claude reports progress against it and the evaluator judges it.

#### Name the evidence

Say which output proves the condition, and that it must appear after the last change.

#### Make it finite

"All tasks done" needs a list of tasks. Name the list or the place it lives.

#### Say what counts as blocked

Tell the agent what to report when it cannot finish, so a blocked goal ends with a report and does not spin.

### Before and after

| Weak | Checkable |
|---|---|
| Fix the flaky test | `test_upload_retry` has passed 20 times in a row, shown by the loop's printed output after the last change; or stop after 10 turns |
| Finish the docs | Every page listed in `docs/index.md` exists and the link checker's printed output shows 0 broken links |
| Work through the backlog | Every in-progress task in the tracker is closed or marked blocked with its reason, shown by the tracker's printed list |

#### Example: a full goal

```text
/goal Every in-progress task in the tracker is either closed with its commit
named, or marked blocked with what unblocks it. Print the tracker's list of
in-progress tasks as the last thing you do; it must be empty or contain only
blocked tasks with reasons. No test file may be edited to make a test pass.
Or stop after 30 turns and report what is left.
```

## A Stop hook as an enforced goal

`/goal` relies on a model reading what the agent printed. A Stop hook can run the check itself:

- "Done" is decided by the check, and not by the agent's account of it.
- The hook lives in a settings file, so it applies to every session in that scope without anyone typing it.
- It is the systems approach applied to finishing: the condition is enforced by a tool, and a written "run the tests before you stop" becomes the second layer.

### How a Stop hook works

1. Claude finishes a turn and is about to stop.
2. Claude Code runs the hook, passing JSON on standard input, including `stop_hook_active` (true when an earlier Stop hook already kept Claude going).
3. The hook decides.
   - Exit code 0: Claude stops.
   - Exit code 2: Claude does not stop; the hook's standard error is given to Claude as the reason, and it keeps working.
4. Claude Code overrides a hook that blocks eight times in a row with no tool call in between; `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` raises the cap.

### Example: do not stop while tests fail

```bash
#!/usr/bin/env bash
# Stop hook: keep working while the export tests fail. Allows the stop on a
# second consecutive block, so a stuck run ends and reports.
input="$(cat)"
if [[ "$(jq -r '.stop_hook_active' <<<"$input")" == "true" ]]; then
  exit 0
fi
if git diff --quiet origin/main -- src/export; then
  exit 0                      # no change in this area since main: no check needed
fi
if ! out="$(pytest tests/export -q 2>&1)"; then
  printf 'tests/export still fails:\n%s\n' "$(tail -n 20 <<<"$out")" >&2
  exit 2
fi
exit 0
```

Registered in `.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      { "hooks": [ { "type": "command", "command": "tools/stop-tests.sh" } ] }
    ]
  }
}
```

| | `/goal` | Stop hook running a script |
|---|---|---|
| Who judges | A small model reading the conversation | Your script |
| Sees | Only printed output | The repository, the tests, anything a script can run |
| Scope | One session, typed each time | Every session in the settings scope |
| Best for | Ad hoc, judgment-shaped conditions | Checks that must always hold |

## `/loop` and scheduled tasks

`/loop` repeats a prompt while the session stays open.

| You type | What happens |
|---|---|
| `/loop 5m check the deploy` | Runs on a fixed schedule; units `s`, `m`, `h`, `d`, rounded to cron's one-minute steps |
| `/loop check whether CI passed and address review comments` | Self-paced: Claude picks a delay between one minute and one hour each time, and says why |
| `/loop` | Runs a built-in maintenance prompt, or your `.claude/loop.md` (project) or `~/.claude/loop.md` (user) |
| `/loop 20m /review-pr 1234` | Re-runs a skill each time, if Claude is allowed to invoke it |

### Uses

| Use | Example |
|---|---|
| Watch a deploy | `/loop 5m check the deploy status and stop when it is live or failed` |
| Follow a pull request | `/loop 30m read new review comments and answer the ones you can` |
| A one-time reminder | `in 45 minutes, check whether the integration tests passed` (plain words, no `/loop`) |

### Stopping a loop

- A self-paced loop: press `Esc` while it waits, or let Claude end it when the work is done.
- A fixed-interval loop: ask Claude to cancel it ("cancel the deploy check loop"); `Esc` does not stop it.
- Any recurring task expires 7 days after creation, after one final run.

### Limits and behaviour

| Item | Value |
|---|---|
| Scheduled tasks per session | Up to 50 |
| When a task fires | Between your turns, never mid-response |
| Missed runs | No catch-up: one run when Claude becomes idle |
| Timing | Recurring tasks run a little late on purpose (jitter), up to half the interval and at most 30 minutes |
| Underlying tools | `CronCreate`, `CronList`, `CronDelete` |
| Turning it off | `CLAUDE_CODE_DISABLE_CRON=1` |

For work that should run without an open session, use cloud scheduled tasks, Desktop scheduled tasks or a scheduled CI job.

<details markdown="1">
<summary>Goal or loop?</summary>

Use a goal when the work has an end you can describe. Use a loop when the work is a check that repeats on a clock, whether or not anything changed. A deploy watch is a loop; finishing a migration is a goal.

</details>

## How goals and loops end

| Ending | `/goal` | `/loop` | Stop hook |
|---|---|---|---|
| Work done | Evaluator says met | Claude ends a self-paced loop | Script exits 0 |
| Cannot be done | Evaluator says impossible | You cancel it | Block cap reached |
| Stalled | No tool calls for several turns: control returns to you, goal still set | 7-day expiry | Block cap reached |
| Error | Auth, credit, unrecoverable context overflow or unavailable model clears it; other errors retry or pause | | |
| You end it | `/goal clear` | `Esc` or ask to cancel | Remove the hook |

In each case, write the end state to the tracker or a notes file, so the next session knows where things stand.

### What a good stop report says

```text
Stopped at the turn limit (30).
Done: 4 of 6 tasks closed (commits named above).
Left: task 5 waits on a credential only a person can renew;
      task 6 fails a test that also fails on main (output above).
Next: renew the credential, then rerun task 5.
```

## Examples from my own harness

- Goal templates, such as "work every task in progress to the end", started and ended through skills.
  - Starting through a skill means the condition always names the tracker's printed list as its evidence.
  - Ending through a skill means every run closes with the same report: what landed, what is blocked, what needs me.

To start: one `/goal` with a clear finish condition, the printed output that proves it, and a turn clause.

## Pitfalls and lessons

| Pitfall | What helps |
|---|---|
| The evaluator accepts a confident summary | Name printed output in the condition, or move the check into a Stop hook |
| A goal with no way to say "blocked" spins until its bound | Put the blocked case in the condition |
| A Stop hook that blocks forever | Check `stop_hook_active`; the block cap is eight |
| A Stop hook that runs the full suite after every turn | Run it only when relevant files changed |
| A loop that reports every check, including "nothing changed" | Report only changes |
| A forgotten fixed loop | It expires after 7 days; cancel it sooner when the work is done |

## Sources

- `/goal`, verdicts, settings and the comparison with `/loop` and Stop hooks: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-09.
- Stop hook behaviour, `stop_hook_active` and the block cap: <https://code.claude.com/docs/en/hooks-guide.md> and <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-09.
- `/loop` and scheduled tasks: <https://code.claude.com/docs/en/scheduled-tasks.md>, verified 2026-11-09.
- Opus 5.5 harness guidance: <https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5>, verified 2026-11-08.

{% include page_nav.html %}
