---
title: "Building a task set"
parent: "Re-testing after a model change"
grand_parent: "Trends and model changes"
nav_order: 1
---

{% include page_nav.html %}

# Building a task set

As of 2026-11-09. Sub-topic of [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}), in [Trends and model changes]({{ '/sections/18-trends/' | relative_url }}).

A task set is the list of tasks you run before and after a change to see whether the change helped. It is what makes a [model re-test]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}) or a [pruning decision]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }}) a measurement. In the [four layers]({{ '/sections/17-building-a-harness/' | relative_url }}#11-the-four-layers), the task set is layer 4 for the harness itself: it validates a change to the harness before the change is kept. This page covers where tasks come from, how to write each one, and how to read the results.

![Sequence diagram of turning a friction entry into a re-test task. You pick an entry from the friction log and write a task with a prompt and a check before any run. A runner loads the task, runs it three times on the model with the same harness, gets the final tree and transcript for each run, and runs the test suite on each; the check returns pass, pass, fail. The runner reports the pass rate with the failing transcript, and you keep the task for the next model change.]({{ '/assets/images/18-trends/task-set-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. One friction entry becomes a task, and the task runs several times with a check on each run. Diagram made with archify.*

## Where tasks come from

| Source | Example | Why it makes a good task |
|---|---|---|
| Your friction log | "Agent said done with a failing test" | It already failed once, so it tests a real weakness |
| A rule in your context file | "Never edit files under generated/" | It tests whether the rule still does its job |
| A task you do every week | "Write the weekly status note" | It tests the work you care about most |
| A model's known habit | "Uses bold lead-ins in summaries" | It tests the rule or check you wrote for that habit |
| A guard you shipped | "Stop hook refuses a finish with untested edits" | It tests that the guard still fires, and only when it should |

Anthropic suggests starting with 20 to 50 simple tasks drawn from real failures.

## Writing one task

### The parts of a task

1. An id.
   - A short name you can read in a results table.
2. A prompt.
   - The exact text the agent receives, written as you would type it.
3. A starting state.
   - The repository commit, files or documents the task starts from, so every run starts the same way.
4. A check.
   - How you tell a good result from a bad one, written before the first run.

### A complete task

```yaml
- id: no-generated-edits
  start: "repo at commit 3f2a9c1"
  prompt: "Rename the field user_name to username everywhere."
  check:
    program: |
      make gen && git diff --quiet -- generated/
  notes: "From the friction log, 2026-11-05: an agent edited generated/ by hand."
```

The check runs `make gen` on the agent's final tree. If the agent edited a file under `generated/` by hand, or changed the schema without regenerating, the regenerated files differ from what the agent left, and `git diff --quiet` fails.

## Checks a program can run, and checks that need a person

| Kind | Examples | How to run it |
|---|---|---|
| Program | Tests pass; a file was not edited; output matches a format | A script, every run |
| Pattern | No em dashes; no bold lead-ins | A script over the text, every run |
| Judgment | The summary reads well; the design choice is sound | A person, or a model grader you have checked against a person, on a sample |

Use program checks where you can. They are cheap to run several times, and they give the same answer every time.

<details markdown="1">
<summary>Using a model as a grader</summary>

A model can grade judgment checks, such as whether a summary covers the request. Before relying on it, grade a sample of results yourself and compare. If you and the grader disagree often, rewrite the grading instructions or keep that check manual.

</details>

## Reading the results

1. Look at the spread across runs.
   - A task that passes two runs out of three on both models has not changed.
   - A task that goes from three of three to zero of three has.
2. Read the transcripts of failing runs.
   - They show whether the agent ignored a rule, misread the task, or hit a guard.
3. Compare per task.
   - An overall pass rate can stay the same while one important task gets worse.

With a small set, expect noise. The table on [Self-improving harness research]({{ '/sections/17-building-a-harness/self-improving-harnesses.html' | relative_url }}#your-sample-is-small) shows how wide the uncertainty is at 20, 50 and 100 tasks.

## Keeping the set current

| Event | Action on the task set |
|---|---|
| The friction log gets a new kind of entry | Add a task for it |
| A new guard ships | Add a task that the guard should catch |
| The behaviour a task tests no longer matters | Retire the task |
| Any run | Keep the results with the date and the model, as the next baseline |

## Examples from my own harness

- My harness has test suites for its own guards and checks.
  - Many of their cases started as entries in a friction log.
- The re-test after a model change draws on two sources.
  - Those test cases.
  - A set of writing tasks for the writing rules.

To start: write three tasks from your friction log in the format above, each with a program check.

## Sources

- Starting with 20 to 50 simple tasks from real failures: Anthropic, <https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents>, not re-checked for this page.

{% include page_nav.html %}
