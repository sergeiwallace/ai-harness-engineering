---
title: "Self-improving harness research"
parent: "The improvement loop"
grand_parent: "Building your own harness"
nav_order: 1
---

{% include page_nav.html %}

# Self-improving harness research

As of 2026-11-09. Sub-topic of [The improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}), in [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}).

The [improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}) is a manual practice that agents gradually take over. Through 2026, researchers and vendors tried to automate it end to end: a system that reads its own traces and rewrites its own prompts, skills or tools. This page summarises what that work found and what it means for a personal harness.

## The common shape

Across the papers and production reports, the recurring architecture is an outer loop that keeps evidence at every step.

![Sequence diagram of one round of a self-improving harness loop. A proposer agent mines one weakness from traces and makes one small edit to a candidate harness; a protected evaluator runs the tasks and regressions, discards a candidate that regresses, and passes a clean one with its score and diff to a person, who approves it and records the lineage, which becomes the new baseline.]({{ '/assets/images/17-building-a-harness/optimizer-round-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. One round of the common outer loop, with the evaluator and the approval outside what the proposer may edit. Diagram made with archify.*

| Step | What it protects against |
|---|---|
| Capture traces | Fixing from memory instead of from what happened |
| Mine one bounded weakness | Large rewrites that change many things at once |
| Small candidate edit | Changes too big to review or attribute |
| Protected task set | The loop "improving" by changing the test |
| Independent authority | The system approving its own change |
| Lineage and rollback | Being unable to undo a change that looked good |

## Representative work

### Karpathy's autoresearch

A small design pattern for one optimisation task. The agent may edit one file, the evaluator is protected, the experiment budget and the metric are fixed, and each change is kept or discarded from repeated trials after a person confirms the setup. The clean separation between what the agent may edit and what judges it is the part most worth copying.

### Self-Harness and HarnessOpt-Bench

Self-Harness mines failures from traces, proposes edits to the harness, and validates them against regressions. HarnessOpt-Bench evaluates how well models do at this kind of harness optimisation, with a trusted held-out evaluation. Both treat the evaluation protocol as the core of the method.

### Production practice

OpenAI described building self-improving agents with Codex for a production domain, where feedback becomes documentation, and rules that need enforcement become tooling. That is the same progression as the [enforcement ladder]({{ '/sections/17-building-a-harness/' | relative_url }}#3-prefer-a-fix-that-a-program-enforces), carried out by agents.

### The counter-result

One study found that automatic harness changes did no better than simply running the model more. Before trusting an automatic loop, compare it against that baseline: the same model, the same budget, more runs and no harness changes.

## What this means for a personal harness

### Protect the evaluator

If the agent can change the evaluator, it can make any change look good. Keep the task set and the scoring script outside what the loop may edit, and review changes to them separately.

### Your sample is small

With one person's work, the task set is small, and scores are noisy. At 100 tasks with a pass rate near 80 percent, the 95 percent confidence interval is roughly plus or minus 8 percentage points. A change that moves the score by 3 points is inside the noise.

| Tasks in the set | Approximate 95% half-width at an 80% pass rate |
|---|---|
| 20 | about 17 points |
| 50 | about 11 points |
| 100 | about 8 points |

So promote changes on deterministic checks and budgets (did the hook block the bad action, did the run stay under its time limit), and keep a person for questions of taste.

<details markdown="1">
<summary>How the table was computed</summary>

The half-width of a normal-approximation interval for a pass rate p over n tasks is 1.96 times the square root of p(1 - p)/n. For p = 0.8: n = 20 gives 0.175, n = 50 gives 0.111, n = 100 gives 0.078. A Wilson interval gives similar values at these sizes.

</details>

### Keep the person on promotion

The reports agree that the system should not approve its own change. In a personal harness, that means an agent may draft the fix, write the test and open the pull request, and you merge it.

## Sources

These were gathered by my own research survey in 2026-09 and are not yet verified against the primary sources for this page, except where marked.

- Karpathy, autoresearch: <https://github.com/karpathy/autoresearch>, not yet verified against the primary source.
- Self-Harness: Harnesses That Improve Themselves: <https://arxiv.org/abs/2606.09498>, not yet verified against the primary source.
- HarnessOpt-Bench: Evaluating LLMs at Harness Optimization: <https://arxiv.org/abs/2608.06301>, not yet verified against the primary source.
- OpenAI, Building self-improving tax agents with Codex: <https://openai.com/index/building-self-improving-tax-agents-with-codex/>, not yet verified against the primary source.
- Automatic harness changes doing no better than running the model more: <https://arxiv.org/abs/2607.12227>, verified 2026-11-09.

{% include page_nav.html %}
