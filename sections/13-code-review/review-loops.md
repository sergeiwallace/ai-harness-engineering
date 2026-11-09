---
title: "Review rounds and stop rules"
parent: "Code review"
nav_order: 3
redirect_from:
  - /sections/13-code-review/advanced.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Review rounds and stop rules
{: .no_toc }

Agent review is cheap enough to run many times, which makes a new failure common: a loop in which each round's reviewer finds something, the author fixes it, and the next reviewer finds something else. The code gains defensive layers, and the loop ends only when someone stops it.

This page covers:

1. why open-ended review loops do not converge
2. three rules that make them converge, with the commands and briefs
3. how to read a recurring finding
4. a review loop described as states with explicit exits
5. pitfalls.

Its sub-topic, [Verifying review findings]({{ '/sections/13-code-review/verifying-findings.html' | relative_url }}), covers checking each finding before anyone fixes it.

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

The kinds of finding a fresh reviewer can always produce:

| Kind | Example | Usually |
|:--|:--|:--|
| Hypothetical edge case | "What if the list has a billion items?" | Optional, unless a criterion names it |
| Defensive check | "Validate that `user` is not null" behind middleware that already does | A false finding (see [verifying findings]({{ '/sections/13-code-review/verifying-findings.html' | relative_url }})) |
| Style or naming | "Rename `items` to `export_items`" | Optional |
| Real defect | "The JSON export drops archived items" | Must-fix |

### A changing target

A review loop has two jobs:

1. Improve the change.
2. Decide when more review is no longer worth it.

A turn cap handles the second as a budget and does nothing for the first. When each round's reviewer sets the standard, the acceptance target changes every round:

- Round 1's reviewer cares about error handling.
- Round 2's reviewer cares about naming and finds the round 1 fixes verbose.
- Round 3's reviewer asks for the error handling round 2 removed.

A clean round then never becomes a stable end point.

### Over-engineering from adversarial review

Evidence from three directions:

- My own work, especially with Codex and GPT models: an open-ended "find problems" review produced more defensive, less clear code with each round.
- Anthropic's Claude Code best-practices page: a reviewer told to hunt for gaps "leads to over-engineering: extra abstraction layers, defensive code, and tests for cases that can't happen."
- The same page still recommends a separate verifier "so the agent doing the work isn't the one grading it."

Together they support independent review with a bounded mandate:

| Bounded mandate | What it looks like in the brief |
|:--|:--|
| Named criteria | "Review against AC1 to AC4" |
| A severity per finding | must-fix, worth considering, optional |
| Permission to report nothing | "No change warranted is a valid result" |

## Three rules for convergence

### 1. Fix the criteria before the first round

Write the acceptance criteria before review starts, and sort every finding against them:

| Finding | Severity | Starts a new round? |
|:--|:--|:--|
| Breaks a criterion | Must-fix | Yes |
| Unsafe (security, data loss) | Must-fix | Yes |
| A real risk the criteria do not cover | Worth considering | No; the owner decides |
| Style, preference, hypothetical | Optional | No |

### 2. Review only the new diff

After the first round, each re-review sees only what changed since the last reviewed state:

```bash
git diff <last-reviewed-sha>..HEAD > review/round-2.patch
```

The round-2 brief then limits the scope:

```text
Review only review/round-2.patch, against the criteria in review/criteria.md.
These changes respond to the round-1 findings in review/round-1.md.
Report a finding outside this patch only if it breaks a criterion.
"No change warranted" is a valid result.
```

- A fresh reviewer then cannot reopen code that round 1 already settled.
- Claude Code's team reports that capping rounds and limiting re-review to the new diff, together with [verifying each finding]({{ '/sections/13-code-review/verifying-findings.html' | relative_url }}), raised the share of internal pull requests that passed after one round from 65% to 78%.

### 3. Stop when findings stop being new

The loop ends when a round produces no new must-fix finding. Three checks keep that honest:

| Signal | What it might mean | Check |
|:--|:--|:--|
| The first round finds nothing | The reviewer did not really read the target | Ask it to cite three specific hunks it read |
| A round finds a sibling of an earlier finding | The fix addressed an instance, not the cause | See [recurring findings](#recurring-findings) |
| The same finding returns after being fixed | Two criteria conflict | Stop and resolve the conflict with the owner |

## Recurring findings

### What the pattern looks like

I had 17 separate agents analyse a corpus of long review loops from my harness. Every one of them reported the same shape: each new round discovered a sibling of an earlier finding. The kinds of sibling:

| Sibling | Example |
|:--|:--|
| A different state | The fix handled "running"; the next round found "paused" |
| Another consumer | The fix covered the CLI; the next round found the API |
| The opposite polarity | The fix handled "flag set"; the next round found "flag unset" |
| Another surface of the same obligation | The fix updated the code; the next round found the doc page |

Fresh reviewers were doing, late, the impact analysis the first fix should have included.

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
| Re-specify | A reviewed, narrower version of the change is ready | Admit |
| Stabilise | Risk-tiered confirmation is satisfied | Promote (merge) |

<!-- IMAGE NEEDED: a state diagram of a review loop with states Admit, Audit, Fix, Confirm, Stabilise, Re-specify, Escalate, Promote and labelled transitions; search: state machine diagram code review loop, review workflow state diagram -->

No single round cap fits every kind of work. Thresholds should vary with risk and be tuned from what actually escapes:

- an escaped defect after a short loop argues for more confirmation on that kind of change
- loops that keep reaching the budget with no blocker argue for re-specifying earlier.

<details markdown="1">
<summary>Aside: what agent frameworks provide</summary>

Most agent frameworks offer a loop limit, which stops runaway execution and does not check whether successive findings share a root cause:

| Framework | Loop limit |
|:--|:--|
| LangGraph | A maximum super-step recursion limit |
| Google ADK | `LoopAgent` requires a termination mechanism |
| CrewAI | `max_iter`, default 20 |
| AutoGen GraphFlow | A reviewer loop with an explicit exit plus `MaxMessageTermination(10)` in its example |

These are as summarised in the research behind this page; check each framework's current documentation before relying on a default.

</details>

## Pitfalls

| Pitfall | Why it bites | Fix |
|:--|:--|:--|
| Fixing every finding from every round | The code grows defensive layers | Only must-fix findings start a new round |
| Re-reviewing the whole change each round | Settled code is reopened | Review only the new diff |
| A turn cap as the only stop rule | The loop ends on budget, not on convergence | Stop when findings stop being new |
| Patching each sibling finding | The same cause keeps producing findings | Fix the invariant and test it |
| A first round with zero findings, accepted as is | The reviewer may not have read the change | Ask for the hunks it read |

## From my harness

How my audit rounds run:

1. Each round sorts findings into must-fix, worth considering, and take it or leave it.
2. Only must-fix findings start another round.
3. Every review brief pairs "look for improvements" with "no change warranted is a valid result".
4. A first round with no findings is checked for whether the reviewer read the target.
5. A round with nothing new ends the loop.

Steps 1, 2 and 5 are built into the audit workflow (layer 2 of the harness); step 3 is a brief line (layer 1).

## Sources

- [Claude Code best practices](https://code.claude.com/docs/en/best-practices.md), verified 2026-11-08.
- [Claude Code code review at Anthropic (65% to 78%)](https://claude.dev/blog/solving-code-review-at-anthropic/), verified 2026-11-09.
- Framework loop limits: as summarised in the research behind this page; not re-verified against each framework's documentation.

{% include page_nav.html %}
