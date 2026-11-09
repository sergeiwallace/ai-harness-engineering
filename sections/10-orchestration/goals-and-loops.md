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

A goal keeps an agent working until a condition is met. A loop repeats a prompt on a schedule. Both turn a single answer into sustained work, and both need an exit condition and a limit. This page covers `/goal` and how its evaluator works, how to write a condition it can judge, `/loop`, limits, and how goals end. The [section overview]({{ '/sections/10-orchestration/' | relative_url }}) is the short version. A sub-topic page covers [hill-climbing and quality ratchets]({{ '/sections/10-orchestration/hill-climbing.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## `/goal`

![A sequence diagram of one goal. You set the goal "suite shows 0 failures". Inside a loop fragment the agent runs the test suite, gets 2 failures printed, ends its turn, and the evaluator, which only reads, answers "not met: continue". The agent fixes and runs again, gets 0 failures printed, ends its turn, and the evaluator answers "condition met". The agent reports done with the output.]({{ '/assets/images/10-orchestration/goal-run.svg' | relative_url }}){: style="width:100%"}

*Figure: a `/goal` run; the evaluator judges only what was printed. Made with archify.*

### How it works

1. You set the goal: `/goal <condition>`.
2. The agent works a turn and ends it.
3. At the end of the turn, a small fast model reads the conversation.
   - It calls no tools.
   - It decides whether the condition is met.
4. If not met, the agent gets another turn; if met, the goal ends.

### What the evaluator can and cannot see

| Can see | Cannot see |
|---|---|
| Everything printed in the conversation | Files on disk |
| Command output the agent showed | Whether a command was really run, if its output was not shown |
| The agent's own claims | Whether those claims are true |

So the condition has to name evidence the agent will print. "The bug is fixed" invites the agent to say "the bug is fixed". "The regression test's output, shown after the last change, reports it passing" asks for something the evaluator can read.

## Writing a condition

A condition the evaluator can judge has three properties. Each has its own sub-section below.

| Property | Question it answers | Missing it looks like |
|---|---|---|
| Names the evidence | What printed output proves it? | "The bug is fixed" |
| Finite | Which list, which set? | "All tasks done" |
| Has a way out | What if it cannot finish? | The loop spins to its limit |

#### Name the evidence

Say what output proves the condition, and that it must appear after the last change.

#### Make it finite

"All tasks done" needs a list of tasks. Name the list or the place it lives.

#### Say what counts as blocked

A goal that cannot finish should end with a report, not spin. Say what the agent should report when it cannot finish.

### Before and after

| Weak | Checkable |
|---|---|
| Fix the flaky test | `test_upload_retry` has passed 20 times in a row, shown by the loop's printed output after the last change |
| Finish the docs | Every page listed in `docs/index.md` exists and the link checker's printed output shows 0 broken links |
| Work through the backlog | Every task in progress in the tracker is closed or marked blocked with its reason, shown by the tracker's printed list |

#### Example: a full goal

```text
/goal Every task in progress in the tracker is either closed with its
commit named, or marked blocked with what unblocks it. Show the tracker's
list of in-progress tasks as the last thing you print; it must be empty
or contain only blocked tasks with reasons.
```

## Limits

Anthropic's guidance for Opus 5.5 recommends the same evaluator pattern for custom harnesses, and stopping after two or three automatic continuations. A limit catches the cases a condition misses: an unreachable condition, an evaluator that keeps saying "not yet", an agent going in circles.

> Every loop needs an exit condition and a limit. When the limit is hit, the agent should stop and report what is done, what is left, and why it stopped.
{: .warning }

### What a good stop report says

```text
Stopped at the continuation limit.
Done: 4 of 6 tasks closed (commits named above).
Left: task 5 waits on a credential only a person can renew;
      task 6 fails a test that also fails on main (output above).
Next: renew the credential, then rerun task 5.
```

## `/loop`

`/loop` repeats a prompt on a time interval, for monitoring or recurring work.

| Use | Example |
|---|---|
| Watch a deploy | `/loop 5m check the deploy status and stop when it is live or failed` |
| Follow a pull request | `/loop 30m read new review comments and answer the ones you can` |
| Recurring chores | `/loop 1h reap finished worktrees and report what was removed` |

A loop's exit is often in the prompt itself ("stop when it is live or failed"); its limit is the session's own life, or an explicit count.

<details markdown="1">
<summary>Goal or loop?</summary>

Use a goal when the work has an end you can describe. Use a loop when the work is a check that repeats on a clock, whether or not anything changed. A deploy watch is a loop; finishing a migration is a goal.

</details>

## How goals end

1. The condition is met: the evaluator says so and the session stops.
2. The limit is hit: the agent writes its stop report.
3. A person ends it: clear the goal, or give a new one.

In each case the end state should be written down, in the tracker or a notes file, so the next session knows where things stand.

## Examples from my own harness

- Goal templates, such as "work every task in progress to the end", started and ended through skills.
  - Starting through a skill means the condition always names the tracker's printed list as its evidence.
  - Ending through a skill means every run closes with the same report: what landed, what is blocked, what needs me.

To start: one `/goal` that has a clear finish condition, naming the printed output that proves it.

## Pitfalls and lessons

| Pitfall | What helps |
|---|---|
| The evaluator accepts a confident summary | Name printed output in the condition |
| A goal with no way to say "blocked" spins until the limit | Put the blocked case in the condition |
| A loop that reports every check, including "nothing changed" | Report only changes |

## Sources

- `/goal`: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-08.
- Opus 5.5 harness guidance: <https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5>, verified 2026-11-08.
- `/loop`: <https://code.claude.com/docs/en/scheduled-tasks.md>, verified 2026-11-08.

{% include page_nav.html %}
