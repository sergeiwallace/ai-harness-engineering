---
title: "Review rounds and stop rules"
parent: "Code review"
nav_order: 3
redirect_from:
  - /sections/13-code-review/advanced.html
---

{% include page_nav.html %}

# Review rounds and stop rules
{: .no_toc }

As of 2026-11-09. Part of [Code review]({{ '/sections/13-code-review/' | relative_url }}).

Agent review is cheap enough to run many times. A common failure is a loop in which each round's reviewer finds something, the author fixes it, and the next reviewer finds something else. The code gains defensive layers, and the loop ends when someone stops it. This page covers why open-ended review does not converge, the three rules that make it converge, how to read a recurring finding, and why adversarial review over-engineers code.

<figure>
  <img src="{{ '/assets/images/13-code-review/review-rounds.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: owner, author agent, reviewer and findings ledger. Round 1 logs five findings and the author fixes two must-fix ones. Round 2 reviews the new diff only and logs one finding that is a sibling of round 1; the owner asks for the root cause to be fixed. Round 3 reviews the new diff and logs zero new findings, so the loop stops.">
  <figcaption>A loop that converges in three rounds. Made with archify.</figcaption>
</figure>

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## Open-ended review loops

### Findings in every round

Claude Code's best-practices page says: "A reviewer that's asked to find problems will always find *something*. If you fix every finding, you end up with extra abstraction layers and tests for cases that can't happen."

### A changing target

A review loop has two jobs: improve the change, and decide when more review is no longer worth it. A turn cap does the second as a budget, but does not make the loop converge. When each round's reviewer sets the standard, the acceptance target changes every round, so a clean round never becomes a stable end point.

### Over-engineering from adversarial review

In my own work, especially with Codex and GPT models, an open-ended "find problems" review produced code that was more defensive and less clear with each round. Anthropic's prompting guidance points the same way: newer models over-react to aggressive wording and tend to over-engineer, including defensive code. Its workflows page still recommends independent agents that review each other's work. Together they support independent review with a bounded mandate.

## Three rules for convergence

### 1. Fix the criteria before the first round

Write the acceptance criteria before review starts, and review against them. A finding is must-fix only if it breaks a criterion or is unsafe. Everything else is optional, and optional findings do not start a new round.

### 2. Review only the new diff

After the first round, each re-review sees only what changed since the last one. That stops a fresh reviewer from re-litigating settled code. Claude Code's team reports that capping rounds and limiting re-review to the new diff, together with [verifying each finding]({{ '/sections/13-code-review/verifying-findings.html' | relative_url }}), raised the share of internal pull requests that passed after one round from 65% to 78%.

### 3. Stop when findings stop being new

The loop ends when a round produces no new must-fix finding. Two checks keep that honest:

| Signal | What it might mean | Check |
|:--|:--|:--|
| The first round finds nothing | The reviewer did not really read the target | Ask it to cite three specific hunks it read |
| A round finds a sibling of an earlier finding | The fix addressed an instance, not the cause | See [recurring findings](#recurring-findings) |
| The same finding returns after being fixed | Two criteria conflict | Stop and resolve the conflict with the owner |

## Recurring findings

### What the pattern looks like

I had 17 separate agents analyse a corpus of long review loops from my harness, and every one of them reported the same shape: each new round discovered a sibling of an earlier finding. A different state, another consumer, the opposite polarity, or another surface of the same obligation. Fresh reviewers were doing late the impact analysis the first fix should have done.

### Worked example

| Round | Finding | Fix |
|:--|:--|:--|
| 1 | The export drops archived items when `--all` is set | Add archived items to the `--all` path |
| 2 | The JSON export drops archived items too | Add them to the JSON path |
| 3 | The API's export endpoint drops them too | Add them to the API |

All three rounds have one cause: there is no single definition of "items to export". At round 2 the better fix was to write the invariant ("every export path uses one item selector"), fix the cause once, and test the invariant.

When a finding recurs as a sibling:

1. Pause the review rounds.
2. Write down the invariant the sibling findings share.
3. Fix the invariant at its root and add a test for it.
   - The test covers every surface the siblings appeared on.
4. Resume review on the new diff.

## Review loop states

The research behind this page proposes a review loop with named states and explicit exits, in addition to a round counter:

| State | Leaves when | Goes to |
|:--|:--|:--|
| Admit | The change and its evidence plan are ready | Audit |
| Audit | A new cluster of findings with budget left | Fix |
| Audit | No blocking finding | Stabilise |
| Audit | The same cluster recurs | Re-specify |
| Audit | The budget is spent | Escalate to the owner |
| Fix | The fix and its closure evidence are complete | Confirm |
| Confirm | The targeted fix is confirmed | Audit |
| Stabilise | Risk-tiered confirmation is satisfied | Promote (merge) |

No single round cap fits every kind of work. Thresholds should vary with risk and be tuned from what actually escapes.

<details markdown="1">
<summary>Aside: what agent frameworks provide</summary>

Most agent frameworks offer a loop limit: a maximum number of steps or iterations, a required termination condition, or a maximum message count for a reviewer loop. These stop runaway execution. None of them checks whether successive findings share a root cause.

</details>

## From my harness

My audit skill reviews in rounds and sorts findings into must-fix, worth considering, and take it or leave it. Only must-fix findings start another round. Review briefs pair the request to look for improvements with "no change warranted is a valid result", and a first round with no findings is checked for whether the reviewer read the target. A round with nothing new ends the loop.

## Sources

- Claude Code best practices: <https://code.claude.com/docs/en/best-practices.md>, verified 2026-11-08.
- Claude Code code review at Anthropic (65% to 78%): <https://claude.dev/blog/solving-code-review-at-anthropic/>, verified 2026-11-09.

{% include page_nav.html %}
