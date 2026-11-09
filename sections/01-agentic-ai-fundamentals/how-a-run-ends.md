---
title: "How an agent run ends"
parent: "The agent loop"
grand_parent: "Agentic AI fundamentals"
nav_order: 1
---

{% include page_nav.html %}

# How an agent run ends

As of 2026-11-09.

An agent run ends in one of four ways: the model says it is done, a person stops it, a limit fires, or a check decides it is not done yet and sends it back. Most of the hard problems in long autonomous work are stopping problems: a run that ends too early with a confident "done", or one that never ends at all. This page is part of [The agent loop]({{ '/sections/01-agentic-ai-fundamentals/the-agent-loop.html' | relative_url }}).

> A final answer records what the model believes about its own work. A completion check tests the stated condition before the run is allowed to end.
{: style="border-left: 4px solid #0f766e; background: #f0fdfa; padding: 0.75rem 1rem;"}

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The four exits

![Diagram: the paths out of an agent run, top to bottom. A task starts the working loop; from working, Esc leads to interrupted and a cap leads to limit reached; when the model judges it done it writes a final answer and tries to stop; a completion check either confirms the condition and the run is done, sends it back to working with a reason, or after its retry cap ends in a blocked report]({{ '/assets/images/01-agentic-ai-fundamentals/run-end-paths.svg' | relative_url }}){: style="width: 100%" }
*Figure: every way a run can end. The model chooses the final answer; the completion check, the limits and you decide whether the run actually stops there.*

| Exit | Who decides | Typical trigger | Risk |
|---|---|---|---|
| Final answer | the model | it judges the goal met, or decides it is blocked | stops early with work undone |
| Interrupt | a person | the run is heading the wrong way | needs someone watching |
| Limit | the client or a wrapper | turns, minutes, money or tokens run out | stops mid-step, state may be half-written |
| Completion check | a hook, an evaluator or a script | the stated condition is not yet true | can loop if the condition is unreachable |

## Exit 1: the final answer

### What it is

When the model writes a reply with no tool call, the client treats it as the end of the task. In the Messages API this is a reply whose stop reason is `end_turn` rather than `tool_use` (see [Messages and content blocks]({{ '/sections/01-agentic-ai-fundamentals/the-agent-loop.html' | relative_url }}#messages-and-content-blocks)).

### Why it can be wrong

#### Premature completion

The model's judgement of "done" rests on what is in its context. If it ran one test and saw it pass, it may report the task complete without running the suite. Nothing in the loop itself stops that.

#### The confident false report

A final answer is prose the model wrote about its own work. It can say "all tests pass" when no test was run in the session, especially after a compaction that dropped the record of what was actually run.

> Check the tool calls in the transcript to see which commands ran and what they printed. The final message records what the model believes it did.
{: style="border-left: 4px solid #b91c1c; background: #fef2f2; padding: 0.75rem 1rem;"}

### Worked example: reporting a blocked task

An agent asked to deploy a service finds the cloud credentials expired. A good final answer says so: "Blocked: the deploy step needs a fresh login; nothing was deployed." A bad one says "Deployment steps complete" because every command it could run did run. The difference is in the instructions: tell agents that "blocked" is an acceptable answer and must be said plainly.

## Exit 2: the interrupt

A person can stop a run at any moment. In Claude Code, Esc stops the current turn and leaves the conversation open, so you can correct course and continue. The model sees the interruption in its context and the partial work stays in place.

Interrupts work only when someone is watching. For long runs, the practical alternatives are limits and checks, which do not need a person at the screen.

## Exit 3: limits

### Kinds of limit

| Limit | Where it is set | What it protects |
|---|---|---|
| Turn limit | a flag on a headless run, or a wrapper's loop counter | runaway loops |
| Time limit | a command timeout, a CI job timeout | hung commands and stuck runs |
| Cost limit | a budget on the account or the wrapper | spend |
| Context limit | the model's window size | the run itself, via compaction |

### The context limit is different

A full context window does not end a Claude Code session. The client compacts the history into a summary and the run carries on. The work continues, but with less detail than it had. [The context window and tokens]({{ '/sections/01-agentic-ai-fundamentals/context-window-and-tokens.html' | relative_url }}#compaction) covers what survives.

### Worked example: a limit with a clean stop

A nightly job runs an agent headless with a turn cap. When the cap fires mid-task, the run ends wherever it was, which can leave a half-edited file. A cleaner design pairs the hard cap with a soft instruction one step earlier: "after 40 turns, stop starting new work, commit what is done, and write what is left to the plan file". The instruction usually produces a clean stop, and the hard cap still ends the run if the instruction is missed.

## Exit 4: completion checks

### The idea

A completion check runs when the model tries to finish and tests the stated condition. The figure at the top of this page shows it as the "not yet" arrow. In order:

1. The model writes a final answer: "done".
2. The client runs the check before ending the turn.
   - A Stop hook runs a script, such as the test suite.
   - An evaluator model reads the condition and the latest turn.
3. The check answers.
   - Condition met: the run ends.
   - Not met: the client returns the reason to the model ("2 tests still fail") and the loop continues.
4. The model works again, then writes a new final answer, and the check runs again.
5. After the check's retry cap, the run stops with a blocked report.

### Three ways to build one

#### A Stop hook

Claude Code runs `Stop` hooks when the agent is about to end its turn. A hook script can run the test suite and, if it fails, block the stop and return a message the model reads, such as "tests failing: fix before finishing". [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) shows how hooks are registered.

#### An evaluator model

Claude Code's `/goal` sets a condition in plain words. After each turn a small model judges whether the condition is met and, if not, the agent continues. This fits conditions that are hard to script, such as "the design doc answers every open question".

#### A wrapper script

A script outside the client runs the agent, checks the result itself (exit codes, files, test output), and starts another run with the failure if needed. This is the most transparent and the easiest to log.

<details markdown="1">
<summary>Choosing between them</summary>

| | Stop hook | Evaluator (`/goal`) | Wrapper script |
|---|---|---|---|
| Condition type | scriptable: tests, lint, file exists | judged in words | scriptable |
| Runs inside the session | yes | yes | no, it starts sessions |
| Cost | the check's runtime | a small model call per turn | a full new run per retry |
| Failure mode | a slow check slows every stop | a lenient judge passes weak work | retries lose the session's context |

A sensible default is a Stop hook for anything a script can measure, and an evaluator only for conditions that need judgement.
</details>

### The unreachable condition

A completion check can create the loop it was meant to prevent. If the condition cannot be met, for example the tests need a service that is down, the agent is sent back again and again.

> Every completion check needs its own exit: a maximum number of returns, after which the run stops and reports what blocks it.
{: style="border-left: 4px solid #b91c1c; background: #fef2f2; padding: 0.75rem 1rem;"}

## A stopping checklist for long runs

1. State the goal as a condition a check can test ("the suite passes", "the file exists"). An activity such as "work on the tests" gives a check nothing to test.
2. Say that "blocked" is an acceptable answer, and what a blocked report must contain.
3. Put a scriptable check at the stop step.
4. Cap the retries of that check.
5. Put a hard limit outside everything: turns, time or money.
6. Make the agent write its state to a file before any planned stop, so the next run can resume (see [Sessions and state]({{ '/sections/01-agentic-ai-fundamentals/sessions-and-state.html' | relative_url }})).

## Sources

- `/goal` and its evaluator: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-09.
- `Stop` hook behaviour: <https://code.claude.com/docs/en/hooks.md>, not yet verified against the primary source for this page's date.
- Stop reasons in the Messages API: <https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview>, not yet verified against the primary source for this page's date.
- The examples and the checklist are written for this page.

{% include page_nav.html %}
