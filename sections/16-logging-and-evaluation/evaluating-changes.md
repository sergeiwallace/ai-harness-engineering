---
title: "Evaluating a harness change"
parent: "Logging, observability and evaluation"
nav_order: 2
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Evaluating a harness change

As of 2026-11-09.

Part of [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}). Sub-page: [pass@k and pass^k]({{ '/sections/16-logging-and-evaluation/pass-at-k.html' | relative_url }}).

Every rule, hook, skill and model choice in a harness is a bet that it makes the agent better. An evaluation turns the bet into a measurement: a fixed set of tasks, run with and without the change, compared on outcomes that matter. Anthropic's guide to agent evaluation frames the object being measured: "When we evaluate 'an agent,' we're evaluating the harness and the model working together."

## Contents

- [The loop](#the-loop)
- [Building the task set](#building-the-task-set)
- [Grading](#grading)
- [Running fairly](#running-fairly)
- [Reading the result](#reading-the-result)
- [Capability and regression sets](#capability-and-regression-sets)
- [Tools](#tools)
- [A worked evaluation](#a-worked-evaluation)
- [Examples from my own harness](#examples-from-my-own-harness)
- [Pitfalls](#pitfalls)

## The loop

![Start from a fixed task set of 20 to 50 real failures. Run a baseline each task k times, change one thing, run again with the same tasks and the same k, compare pass rate, cost and time, then keep or revert the change before the next one.]({{ '/assets/images/16-logging-and-evaluation/eval-loop.svg' | relative_url }}){: style="width:100%"}

*The evaluation loop. One change per pass round the loop.*

![Sequence diagram of the same loop as it runs: the runner gives each task to the agent with the old harness and then the new one, grades each result, and records pass, cost and time in the ledger before reporting keep or revert.]({{ '/assets/images/16-logging-and-evaluation/eval-run-sequence.svg' | relative_url }}){: style="width:100%"}

*What happens during one evaluation, step by step.*

## Building the task set

### Start from real failures

The best tasks are ones your agents actually got wrong. Each friction log entry, reverted commit or correction you typed twice is a candidate. Anthropic suggests starting with 20 to 50 simple tasks drawn from real failures; a small set you run often beats a large one you never run.

### What a task contains

| Part | Example |
|---|---|
| Starting state | a repository at a fixed commit |
| Instruction | "Add input validation to the signup form" |
| Success check | the hidden test file passes; no other test fails |
| Budget | a time limit and a cost limit |

### Freeze it

Pin the starting commit and the instruction text. A task set that changes between runs cannot tell you whether the harness changed the result.

### Example: three tasks from one week's friction

```yaml
- id: signup-validation
  repo_commit: 4f2a9c1
  instruction: "Reject signup emails without an @ and show the error under the field."
  check: "npm test -- signup.validation"
- id: export-large-file
  repo_commit: 4f2a9c1
  instruction: "Make the export endpoint handle files over 2 GB."
  check: "pytest tests/test_export_large.py"
- id: flaky-date-test
  repo_commit: 9b81d07
  instruction: "Fix the flaky date test without skipping it."
  check: "pytest tests/test_dates.py --count=20"
```

## Grading

### Code checks first

Where a task's success can be tested by code (a test passes, a file exists, a command exits zero), use that. It is cheap, fast and consistent.

### Rubrics for the rest

Some outcomes need judgement: is the summary accurate, is the review comment useful. Write a rubric with a few concrete criteria and have a model grade against it, and check a sample of its grades by hand to make sure the grader agrees with you.

### Grade the outcome

An agent that reaches a correct result by a route you did not expect should pass. Grading the exact steps penalises improvements.

## Running fairly

#### Change one thing

Hold the model, the task set and the number of runs fixed, and change only the harness element under test. Two changes at once tell you about neither.

#### Run each task several times

Agents are not deterministic. One run per task measures luck. Run each task k times, for example three to five, and report the rate. [pass@k and pass^k]({{ '/sections/16-logging-and-evaluation/pass-at-k.html' | relative_url }}) explains which rate to report.

#### Isolate runs

Each run starts from a clean copy of the starting state, in its own worktree or container, so one run's leftovers do not help or hurt the next.

#### Record everything

Log pass or fail, cost, time and the full transcript of each run in the ledger. The transcript is how you find out why a task failed.

## Reading the result

| Outcome | Reading |
|---|---|
| Pass rate up, cost flat | keep |
| Pass rate up, cost up a lot | keep only if the gain is worth the cost |
| Pass rate flat, cost down | keep: same results, cheaper |
| Pass rate flat, cost up | revert: the change only costs |
| Pass rate down | revert |

> With 30 tasks, a difference of one or two passes is within run-to-run variation. Grow the task set before trusting a small gain.

<details markdown="1">
<summary>Aside: how big a difference is real?</summary>

With 30 tasks run three times each, you have 90 graded runs per side. A change from 60 to 63 passes is well within run-to-run variation for a non-deterministic agent. A change from 60 to 75 is unlikely to be noise. If you need to detect small effects, you need more tasks, not more decimal places.

</details>

## Capability and regression sets

Anthropic's guide separates two purposes. Capability evals find what the agent can do: tasks it fails today, to see whether a change unlocks them. Regression evals stop it from losing ground: tasks it passes today, which must keep passing.

Keep them as two sets and report them separately. A change that solves two new capability tasks and breaks three regression tasks is not an improvement, and a single combined score would hide that.

## Tools

### `claude plugin eval`

Claude Code's `claude plugin eval` compares a plugin's results against a run without it. It is the same loop, packaged for one kind of harness change.

### OpenAI's eval-driven development

OpenAI recommends eval-driven development and running evals on every change: write the evaluation before or with the change, and treat a regression as a failing test.

### Your own runner

A runner can be a short script: for each task and each repeat, create a clean worktree, start the agent with the instruction, run the check, append a line to the ledger.

## A worked evaluation

#### The change

A new hook refuses commits when focused tests have not run since the last edit.

#### The set

30 tasks from the last month's friction log, each run three times with and without the hook, on the same model.

#### The result (illustrative numbers)

| | Without hook | With hook |
|---|---|---|
| Runs passed (of 90) | 58 | 71 |
| Median time per task | 6.1 min | 7.0 min |
| Median cost per task | $0.42 | $0.47 |
| Regression set (20 tasks, 60 runs) | 57 | 58 |

#### The decision

Keep the hook. Pass rate rose by 13 runs, the regression set held, and the cost and time increases are small. Record the result in the ledger next to the change.

## Examples from my own harness

- Changes to the harness are tried on a fixed model first, and rules that compensated for an older model are re-tested when the model changes.
- A plugin's value is checked against a run without it before it is turned on by default.

To start: ten tasks taken from your own past failures, run before and after one harness change.

## Pitfalls

- Without a baseline, you cannot tell whether a harness change helped.
- A task set built from the cases the change was designed for will always show a gain. Include tasks the change was not aimed at.
- Public benchmarks have errors; about 30 percent of one benchmark's tasks were reported broken (not yet verified against the primary source). Prefer your own tasks for your own harness.

## Sources

- Anthropic on agent evals: <https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents> (2026-01-09), verified 2026-11-08.
- OpenAI evaluation best practices: <https://developers.openai.com/api/docs/guides/evaluation-best-practices>, verified 2026-11-08.
- `claude plugin eval`: <https://code.claude.com/docs/en/plugin-evals.md>, verified 2026-11-08.
- The broken-benchmark figure: <https://openai.com/index/separating-signal-from-noise-coding-evaluations/>; the page returned HTTP 403 to a fetch on 2026-11-09, so the figure is not yet verified.

{% include page_nav.html %}
