---
title: "pass@k and pass^k"
parent: "Evaluating a harness change"
grand_parent: "Logging, observability and evaluation"
nav_order: 1
redirect_from:
  - /sections/16-logging-and-evaluation/advanced.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# pass@k and pass^k

Part of [Evaluating a harness change]({{ '/sections/16-logging-and-evaluation/evaluating-changes.html' | relative_url }}), in [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}).

An agent that runs the same task several times does not always get the same result. Two measures from Anthropic's eval guide describe repeated attempts:

- pass@k counts a task as solved if any of k attempts succeeds.
- pass^k counts it as solved only if all k attempts succeed.

The same agent can look excellent under one and poor under the other, so pick the one that matches how you use the agent.

## Contents

- [Definitions](#definitions)
- [The arithmetic](#the-arithmetic)
- [Which one to use](#which-one-to-use)
- [Estimating from runs](#estimating-from-runs)
  - [Average per task, not over the pooled runs](#average-per-task-not-over-the-pooled-runs)
  - [Choosing n and k](#choosing-n-and-k)
- [Worked examples](#worked-examples)
- [Pitfalls](#pitfalls)

## Definitions

| Measure | Solved when | Models the case where |
|---|---|---|
| pass@k | at least one of k attempts succeeds | you can try several times and keep the best, for example with a test that tells you which attempt worked |
| pass^k | every one of k attempts succeeds | the agent must get it right every time, for example a task that runs unattended every day |

## The arithmetic

If one attempt succeeds with probability p, and attempts are independent:

```text
pass@k = 1 - (1 - p)^k      the chance that not all k attempts fail
pass^k = p^k                the chance that all k attempts succeed
```

For p = 0.7:

| k | pass@k | pass^k |
|---|---|---|
| 1 | 0.70 | 0.70 |
| 2 | 0.91 | 0.49 |
| 3 | 0.97 | 0.34 |
| 4 | 0.99 | 0.24 |
| 5 | 1.00 (0.998) | 0.17 |

![Bar chart for a per-attempt success rate of 0.7. For k from 1 to 5, pass@k rises 0.70, 0.91, 0.97, 0.99, 1.00, while pass^k falls 0.70, 0.49, 0.34, 0.24, 0.17.]({{ '/assets/images/16-logging-and-evaluation/pass-at-k.svg' | relative_url }}){: style="width:100%"}

*One agent, two stories. Retries make it look reliable; consistency requirements make it look fragile.*

## Which one to use

| Measure | Use when | Example |
|---|---|---|
| pass@k | a check tells you which attempt worked, so you can keep the best | a test suite picks the passing patch out of three |
| pass^k | the agent runs unattended, or its first answer is trusted | a nightly job; a customer-facing agent |
| Both | comparing two harness versions | report both, side by side |

A change that raises pass@3 and lowers pass^3 made the agent more capable and less consistent. Whether that helps depends on which row above describes how you run it.

## Estimating from runs

Real attempts are not perfectly independent, so estimate from what you observe. Two ways:

- From an overall success rate: quick, and good enough for a first reading.
- Per task, from that task's own runs: closer to the truth for small samples.

### From n runs with c successes

1. Estimate p as c / n.
2. Plug it into the formulas above.
   - 10 runs, 7 successes: p = 0.7, pass@3 = 0.97, pass^3 = 0.34.

### Directly, per task

With n runs per task (n at least k), you can estimate pass^k per task as the fraction of size-k subsets of the runs that are all successes, and pass@k as one minus the fraction that are all failures. Average over tasks.

```python
from math import comb

def pass_at_k(n, c, k):
    """Chance that at least one of k runs drawn from n (c successes) succeeds."""
    if n - c < k:
        return 1.0
    return 1 - comb(n - c, k) / comb(n, k)

def pass_hat_k(n, c, k):
    """Chance that all k runs drawn from n (c successes) succeed."""
    return comb(c, k) / comb(n, k)

print(pass_at_k(10, 7, 3), pass_hat_k(10, 7, 3))   # 0.992, 0.292
```

<details markdown="1">
<summary>Aside: why the per-task estimate differs from the formula</summary>

With 10 runs and 7 successes:

| Method | pass^3 |
|---|---|
| Formula, p = 0.7 | 0.343 |
| Subset estimate | 0.292 |

The subset estimate samples without replacement from the runs you actually have, which reads a small sample more closely. The gap shrinks as n grows.

</details>

### Average per task, not over the pooled runs

Tasks differ in difficulty, and pooling them hides it. Two tasks, each run many times:

| Task | p | pass^3 |
|---|---|---|
| A, easy | 1.0 | 1.000 |
| B, hard | 0.4 | 0.064 |
| Mean of the per-task values | | 0.532 |
| From the pooled p of 0.7 | 0.7 | 0.343 |

- The pooled figure says the agent gets three in a row about a third of the time.
- In fact it always succeeds on A and almost never on B.

Report per-task values, or their mean, and look at the hard tasks by name.

### Choosing n and k

| Choice | Guidance |
|---|---|
| k | match it to the use: 1 for a single trusted answer, the retry budget for pass@k, the run length you need clean for pass^k (7 for a week of nightly runs) |
| n, runs per task | at least k; more runs give a steadier estimate, at the cost of a longer and more expensive evaluation |

## Worked examples

### Example 1: a nightly job

| Step | Value |
|---|---|
| Task | update a dependency report every night |
| Test runs | 20, of which 18 succeed: p = 0.9 |
| pass^7, a clean week | 0.9^7 = about 0.48 |
| Reading | a failure-free week is close to a coin flip |
| Fix in the system | the scheduler retries a failed run once and alerts on a second failure |

### Example 2: two harness versions

| | Version A | Version B |
|---|---|---|
| p per attempt | 0.6 | 0.75 |
| pass@3 | 0.94 | 0.98 |
| pass^3 | 0.22 | 0.42 |

Version B improves both, but the reliability gain (pass^3 nearly doubles) is much larger than the capability gain. For an unattended agent, that is the number to report.

## Pitfalls

- Reporting pass@k for an agent that runs unattended makes it look more reliable than it is. Use pass^k for that case.
- Computing pass^k from a pooled success rate hides the hard tasks. Average per task.
- Runs that share state, such as a cache or a dirty working tree, are not independent. Start each run from a clean copy.
- Comparing pass@3 for one version with pass@5 for another mixes two values of k. Fix k before the runs.

## Sources

- pass@k, pass^k, capability and regression evals: Anthropic, [Anthropic, Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (2026-01-09), verified 2026-11-08.
- The per-task pass@k estimator is the unbiased estimator introduced for code-generation evaluation in Chen et al., ["Evaluating Large Language Models Trained on Code"](https://arxiv.org/abs/2107.03374) (2021), not re-checked for this page. The pass^k analogue and the other formulas on this page are standard combinatorics.

{% include page_nav.html %}
