---
title: "pass@k and pass^k"
parent: "Evaluating a harness change"
grand_parent: "Logging, observability and evaluation"
nav_order: 1
redirect_from:
  - /sections/16-logging-and-evaluation/advanced.html
---

{% include page_nav.html %}

# pass@k and pass^k

As of 2026-11-09.

Part of [Evaluating a harness change]({{ '/sections/16-logging-and-evaluation/evaluating-changes.html' | relative_url }}), in [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}).

An agent that runs the same task several times does not always get the same result. Two measures from Anthropic's eval guide describe repeated attempts. pass@k counts a task as solved if any of k attempts succeeds. pass^k counts it as solved only if all k attempts succeed. The same agent can look excellent under one and poor under the other, so pick the one that matches how you use the agent.

## Contents

- [Definitions](#definitions)
- [The arithmetic](#the-arithmetic)
- [Which one to use](#which-one-to-use)
- [Estimating from runs](#estimating-from-runs)
- [Worked examples](#worked-examples)

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

> At k = 1 the two measures are equal. They diverge as k grows.

## Which one to use

### pass@k for exploration with a verifier

When a check can tell you which attempt worked (a test suite, a compiler, a screenshot comparison), you can run several attempts and keep the one that passes. pass@k is the chance that at least one is usable. This is how capability evals are often reported.

### pass^k for reliability

When the agent runs unattended, or a person will trust its first answer, every attempt has to be right. pass^k is the chance of k correct runs in a row. This is the measure for a scheduled job, a customer-facing agent, or a harness rule you want to hold every time.

### Both, side by side

Report both when you compare harness changes. A change that raises pass@3 and lowers pass^3 made the agent more capable and less consistent; whether that is good depends on how you run it.

## Estimating from runs

Real attempts are not perfectly independent, so estimate from what you observe rather than only from the formula.

### From n runs with c successes

The simplest estimate of p is c / n. Plug it into the formulas above for a quick reading.

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

With 10 runs and 7 successes, the formula with p = 0.7 gives pass^3 = 0.343, while the subset estimate gives 0.292. The subset estimate samples without replacement from the runs you actually have, which is the more honest reading of a small sample. The gap shrinks as n grows.

</details>

## Worked examples

### Example 1: a nightly job

An agent updates a dependency report every night. On 20 test runs it succeeds 18 times, p = 0.9. pass^7, the chance of a clean week, is 0.9^7, about 0.48. A failure-free week is a coin flip. The team adds a check that retries a failed run once and alerts on a second failure.

### Example 2: two harness versions

| | Version A | Version B |
|---|---|---|
| p per attempt | 0.6 | 0.75 |
| pass@3 | 0.94 | 0.98 |
| pass^3 | 0.22 | 0.42 |

Version B improves both, but the reliability gain (pass^3 nearly doubles) is much larger than the capability gain. For an unattended agent, that is the number to report.

## Sources

- pass@k, pass^k, capability and regression evals: Anthropic, <https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents> (2026-01-09), verified 2026-11-08.
- The per-task estimator for pass@k follows the unbiased estimator common in code-generation evaluation; the formulas on this page are standard combinatorics.

{% include page_nav.html %}
