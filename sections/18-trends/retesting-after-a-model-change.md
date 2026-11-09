---
title: "Re-testing after a model change"
parent: "Trends and model changes"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/18-trends/detail.html
  - /sections/18-trends/advanced.html
last_updated: "2026-11-09"
talk_stop: "13"
---

{% include page_nav.html %}

# Re-testing after a model change

Most rules in a harness exist because of something the model got wrong. A new model gets different things wrong. Some rules stop helping, some start to hurt, and some failures appear that no rule covers. This page is a procedure for finding out which, using the same tools as [evaluating any harness change]({{ '/sections/16-logging-and-evaluation/' | relative_url }}).

![Sequence diagram of re-testing a harness after a model change: run the same task set several times on the old and the new model, read the outputs and transcripts for idle rules and new failures, propose a change, approve it, then edit the rules and record the model and date.]({{ '/assets/images/18-trends/model-change-retest-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. The re-test as one exchange between you, an agent, both models, the task set and the rules. Diagram made with archify.*

## When to re-test

| Trigger | Why it matters |
|---|---|
| You switch to a new model on purpose | The obvious case |
| The client changes its default model | The harness now runs against a model nobody chose |
| A model version under the same name is updated | Behaviour can shift without a name change |
| You change reasoning effort or a similar setting | The same model can behave like a different one, and more effort is not always better (see below) |

> Note: in LangChain's Terminal Bench 2.0 runs with gpt-5.2-codex, the `xhigh` reasoning setting on its own scored 53.9%, against 63.6% at `high`, "due to agent timeouts". Their best run used `xhigh` to plan and to verify, and `high` to build. A re-test after an effort change is how you find a result like this in your own harness.

## The procedure

### 1. Keep a task set

A task set is a small list of real tasks with a way to tell whether each one went well. Draw it from your [friction log]({{ '/sections/17-building-a-harness/' | relative_url }}#21-keep-a-friction-log): each past failure becomes a task. Anthropic suggests starting with 20 to 50 simple tasks drawn from real failures. [Building a task set]({{ '/sections/18-trends/building-a-task-set.html' | relative_url }}) covers how to write and read one.

```yaml
# tasks.yaml: one entry per past failure
- id: tests-after-last-edit
  prompt: "Add input validation to parse_date and finish when done."
  check: "the test suite passes on the final tree"
- id: no-generated-edits
  prompt: "Rename the field user_name to username everywhere."
  check: "no file under generated/ was edited by hand; make gen was run"
- id: plain-summary
  prompt: "Summarise this design doc in 150 words for a reader outside the team."
  check: "no em dashes, no bold lead-ins, no three-part lists"
```

#### Two kinds of check

Some checks a program can run, such as whether the tests pass or whether a file changed. Others need judgment, such as whether a summary reads well. Run the first kind automatically, and read a sample of the second kind yourself.

### 2. Run both models on the same tasks

Keep everything else fixed: the same harness, the same tasks, the same settings. Run each task several times on each model, because a single run is noisy.

```bash
# Illustrative: run each task three times on each model, with the same harness.
for model in old-model new-model; do
  for run in 1 2 3; do
    ./run-task-set.sh --model "$model" --tasks tasks.yaml --out "results/$model/run-$run"
  done
done
```

### 3. Read the differences

Look for three things:

1. Rules that never fire with the new model. They may be idle because the failure is gone.
2. Rules the new model follows so literally that results get worse.
3. New failures that no rule covers.

#### Read transcripts as well as scores

A pass rate shows that something changed. The transcripts show what changed, and whether the agent read a rule, followed it, or worked around it. With a small task set, the transcripts are also the more reliable signal: see the noise table on [Self-improving harness research]({{ '/sections/17-building-a-harness/self-improving-harnesses.html' | relative_url }}#your-sample-is-small).

### 4. Change one thing at a time

1. For each idle rule, remove it and re-run the tasks it covers.
   - Keep the removal if nothing regresses.
2. For each new failure, put the fix into the system.
   - A workflow step, a hook or a test where a program can detect the failure; see [the four layers]({{ '/sections/17-building-a-harness/' | relative_url }}#11-the-four-layers).
   - A short context-file line only where it cannot, or as a pointer to the new guard.
3. Re-run the task set after each change, so each result belongs to one change.

### 5. Record what changed and why

```text
2026-10-14  model: new-model
  removed: "Avoid em dashes" from the always-loaded rules; kept it in the writing checklist only
  reason:  0 hits in 30 runs on the new model, 11 hits in 30 on the old
  added:   a pattern check for bold lead-in labels (a line starting with **Word.**), run before publishing
           plus one line in the writing checklist that names the check
  reason:  appeared in 9 of 30 new-model summaries, 1 of 30 old
```

The numbers in this record are illustrative. Keeping a record like it means the next model change starts from a written record.

## A worked comparison

The writing-tells change from the [overview]({{ '/sections/18-trends/' | relative_url }}#worked-example-writing-tells), taken through the procedure:

| Step | What happened |
|---|---|
| Trigger | The model changed |
| Task set | Ten writing tasks: summaries, status notes, a pull request description |
| Run | Each task on both models, several times |
| Differences | Em dashes nearly gone; bold lead-ins, three-part lists and hedges more frequent |
| Change | Kept the em dash rule short; added a pattern check for bold lead-ins and hedge words, and checklist lines for three-part lists, which need a reader |
| Record | One entry per rule, with the model and the date |

<details markdown="1">
<summary>Why the old rule stayed, in a shorter form</summary>

The em dash rule rarely fires on the new model, but a rare failure in public writing is still costly, and the rule is one line. Retiring a rule is right when it costs more than the failure it prevents. Here the cost was small, so it stayed, shortened.

</details>

## Pitfalls

> Warning: changing the harness and the model in the same step makes the result unreadable. If the score moves, you cannot tell which change moved it.

<details markdown="1">
<summary>Trusting a score from one run</summary>

Model output varies from run to run. A difference between one run on each model is often noise. Run several times and look at the spread.

</details>

<details markdown="1">
<summary>Re-testing only the rules you remember</summary>

Forgotten rules are the likeliest to be out of date. Re-test from the inventory described in [Complexity creep and pruning]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }}#1-inventory), not from memory.

</details>

## Examples from my own harness

- When the model changed, I read a batch of fresh output against my writing rules.
  - I found both idle rules and uncovered patterns, as in the worked comparison.
- The writing rules now carry a note of which model each pattern applies to.
  - The next model change starts with a list of rules to re-check.

To start: write five tasks from past failures into a file today, so the next model change has something to run against.

## Sources

- Reasoning-effort results on Terminal Bench 2.0: LangChain, "Improving Deep Agents with harness engineering", 2026-02-17, <https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering>, verified 2026-11-09.
- Starting with 20 to 50 simple tasks from real failures: Anthropic, <https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents>, not re-checked for this page.
- From the author's own harness, described conceptually.

{% include page_nav.html %}
