---
title: "Evaluating a harness change"
parent: "Logging, observability and evaluation"
nav_order: 2
has_children: true
has_toc: false
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Evaluating a harness change

Part of [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}). Sub-page: [pass@k and pass^k]({{ '/sections/16-logging-and-evaluation/pass-at-k.html' | relative_url }}).

Every rule, hook, skill and model choice in a harness is a bet that it makes the agent better. An evaluation turns the bet into a measurement: a fixed set of tasks, run with and without the change, compared on outcomes that matter. Anthropic's guide to agent evaluation frames the object being measured: "When we evaluate 'an agent,' we're evaluating the harness and the model working together."

## Contents

- [The loop](#the-loop)
- [When to evaluate, and what it costs](#when-to-evaluate-and-what-it-costs)
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

1. Fix the task set, the model and the number of runs per task.
2. Run the baseline with the current harness.
3. Change one thing.
4. Run the same tasks, the same number of times.
5. Compare pass rate, cost and time, separately for capability and regression tasks.
6. Keep the change or revert it, and record the result next to the change.

![Start from a fixed task set of 20 to 50 real failures. Run a baseline each task k times, change one thing, run again with the same tasks and the same k, compare pass rate, cost and time, then keep or revert the change before the next one.]({{ '/assets/images/16-logging-and-evaluation/eval-loop.svg' | relative_url }}){: style="width:100%"}

*The evaluation loop. One change per pass round the loop.*

![Sequence diagram of the same loop as it runs: the runner gives each task to the agent with the old harness and then the new one, grades each result, and records pass, cost and time in the ledger before reporting keep or revert.]({{ '/assets/images/16-logging-and-evaluation/eval-run-sequence.svg' | relative_url }}){: style="width:100%"}

*What happens during one evaluation, step by step.*

## When to evaluate, and what it costs

| Change | Evaluate? |
|---|---|
| A new hook, gate or workflow step that every task passes through | yes |
| A new model, or a new model version | yes, across the whole set |
| A plugin or skill you plan to turn on by default | yes |
| A wording fix in one skill used by one task type | a few tasks of that type |
| A typo in a comment | no |

### How far a harness change can move a score

LangChain reported a measured case on Terminal Bench 2.0 with the model held fixed at `gpt-5.2-codex` (not a Claude model) and only the harness changed:

| Run | Score | What changed |
|---|---|---|
| Starting harness | 52.8% | none |
| Improved harness | 66.5% | harness changes only, including a check that asks the agent to verify its work before it exits |
| Reasoning effort `xhigh` throughout | 53.9% | effort only; LangChain attributes the low score to agent timeouts |
| Reasoning effort `high` throughout | 63.6% | effort only |

Two readings for your own evaluations:

- The 13.7-point gain came from the harness alone, with the model unchanged, so harness changes need the same measurement as model changes.
- More reasoning effort scored lower here because runs timed out, so time limits and effort settings belong in the fixed conditions of every run. LangChain used an `xhigh`-`high`-`xhigh` "reasoning sandwich" as its baseline: `xhigh` to plan and to verify, `high` to implement.

An evaluation is itself a paid job. 30 tasks, 3 runs each, before and after, is 180 agent runs. At an illustrative $0.45 per run that is about $81, so run it through the same [budget check]({{ '/sections/16-logging-and-evaluation/spend-and-budgets.html' | relative_url }}) as any other batch.

## Building the task set

### Start from real failures

The best tasks are ones your agents actually got wrong. Sources:

- entries in your friction log;
- reverted commits and the reason for each revert;
- corrections you typed more than once;
- review comments that caught a real bug.

Anthropic suggests starting with 20 to 50 simple tasks drawn from real failures. A small set you run often tells you more than a large one you never run.

### What a task contains

| Part | Example |
|---|---|
| Starting state | a repository at a fixed commit |
| Instruction | "Add input validation to the signup form" |
| Success check | the hidden test file passes; no other test fails |
| Budget | a time limit and a cost limit |

### Freeze it

A task set that changes between runs cannot tell you whether the harness changed the result. Pin:

- the starting commit of each task;
- the instruction text, word for word;
- the check command;
- the model and its settings, unless the model is what you are testing.

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

| Grader | Use when | Example | Watch for |
|---|---|---|---|
| Code check | success can be tested by code | a hidden test file passes; a command exits 0 | a check that tests the wrong thing |
| Rubric, graded by a model | the outcome needs judgement | "the summary names all three causes" | disagreement with a human; check a sample by hand |
| Human review | a small number of high-stakes tasks | a design write-up | slow, so keep it for the few |

### Grade the outcome

An agent that reaches a correct result by a route you did not expect should pass. Grading the exact steps penalises improvements, so check the end state:

| Grade on | Example check |
|---|---|
| Tests | `pytest tests/test_export_large.py` passes and no other test fails |
| Files | the export file exists and has the expected header row |
| Behaviour | the endpoint returns 200 for a 3 GB file within the time limit |
| The steps taken | avoid, unless a step is itself a requirement, such as "never push to main" |

## Running fairly

1. Change one thing.
   - Hold the model, the task set and the number of runs fixed.
   - Two changes at once tell you about neither.
2. Run each task several times.
   - Agents are not deterministic, so one run per task measures luck.
   - Run each task k times, for example three to five, and report the rate; [pass@k and pass^k]({{ '/sections/16-logging-and-evaluation/pass-at-k.html' | relative_url }}) explains which rate.
3. Isolate runs.
   - Each run starts from a clean copy of the starting state, in its own worktree or container.
4. Record everything.
   - Pass or fail, cost, time and the full transcript, in the ledger.
   - The transcript is how you find out why a task failed.

## Reading the result

| Outcome | Reading |
|---|---|
| Pass rate up, cost flat | keep |
| Pass rate up, cost up a lot | keep only if the gain is worth the cost |
| Pass rate flat, cost down | keep: same results, cheaper |
| Pass rate flat, cost up | revert: the change only costs |
| Pass rate down | revert |

<details markdown="1">
<summary>Aside: how big a difference is real?</summary>

With 30 tasks run three times each, you have 90 graded runs per side. A change from 60 to 63 passes is well within run-to-run variation for a non-deterministic agent. A change from 60 to 75 is unlikely to be noise. If you need to detect small effects, you need more tasks, not more decimal places.

</details>

## Capability and regression sets

Anthropic's guide separates two purposes:

| Set | Holds | Answers |
|---|---|---|
| Capability | tasks the agent fails today | does the change unlock them? |
| Regression | tasks it passes today | does it keep passing them? |

Report them separately. A change that solves two capability tasks and breaks three regression tasks would look neutral in a combined score.

## Tools

<!-- IMAGE NEEDED: the report from a `claude plugin eval` run, showing a plugin's results against a run without it; search: "claude plugin eval report", "Claude Code plugin evals" -->

| Tool | What it does | When to use it |
|---|---|---|
| `claude plugin eval` | compares a plugin's results against a run without it | before turning a plugin on by default |
| Eval-driven development (OpenAI) | evaluations written with the change and run on every change | any harness change you will keep |
| Your own runner | for each task and repeat: clean worktree, start the agent, run the check, append to the ledger | your own task set |

A minimal runner loop:

```bash
for task in tasks/*.yaml; do
  for run in 1 2 3; do
    ./run_task.sh "$task" "$run" >> logs/eval.jsonl
  done
done
```

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
- A grader that is part of the harness under test can move with the change. Keep the grader and the task set outside the thing being changed.
- A task that leaks its answer, such as a hidden test the agent can read in the repository, measures copying. Keep the success check out of the agent's reach.

## Sources

- Anthropic on agent evals: [Anthropic, Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (2026-01-09), verified 2026-11-08.
- OpenAI evaluation best practices: [OpenAI, Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices), verified 2026-11-08.
- `claude plugin eval`: [Claude Code docs: Plugin evals](https://code.claude.com/docs/en/plugin-evals.md), verified 2026-11-08.
- [LangChain, "Improving deep agents with harness engineering"](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering) (2026-02-17): 52.8% to 66.5% on Terminal Bench 2.0 with `gpt-5.2-codex` fixed, and 53.9% at `xhigh` against 63.6% at `high`, "due to agent timeouts", verified 2026-11-09. Found through ShipWithAI's "The Complete Claude Code Harness Engineering Guide", which omits the model name.
- The broken-benchmark figure: [OpenAI, Separating signal from noise in coding evaluations](https://openai.com/index/separating-signal-from-noise-coding-evaluations/); the page returned HTTP 403 to a fetch on 2026-11-09, so the figure is not yet verified.

{% include page_nav.html %}
