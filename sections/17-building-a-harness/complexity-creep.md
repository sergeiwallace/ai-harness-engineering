---
title: "Complexity creep and pruning"
parent: "Building your own harness"
nav_order: 3
---

{% include page_nav.html %}

# Complexity creep and pruning

As of 2026-11-09. Topic page in [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}). Slide: [Stop 12, "Complexity creep"]({{ '/talk/' | relative_url }}#stop-12-5-minutes).

Every fix in the [improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}) adds a mechanism: a line in a context file, a hook, a script, a gate. Each one is reasonable on its own. Together they can grow into a patchwork that nobody can hold in their head, where a change in one place breaks another and the agent spends a growing share of its context on rules instead of the task. This page is about noticing that and pruning on the same loop.

## How complexity creeps in

### The four usual paths

| Path | What it looks like | Why it happens |
|---|---|---|
| Overlap | Two rules, or a rule and a hook, cover the same failure | Each was added for a different incident, months apart |
| Out-of-date rules | A rule compensates for a weakness the current model no longer has | Nobody re-tested after the model changed |
| Over-broad guards | A hook fires on correct work and the agent learns to work around it | The first version matched too much |
| Tangled layers | Reusable parts and personal settings live in the same file | It was quicker to add a line where you already were |

### What it costs

#### Context

Every always-loaded line takes room that the task could use. Long, busy context also makes each instruction less likely to be followed, so a large rule set can lower adherence to the rules that matter.

#### Time

Every hook runs on every matching event. A slow one adds its delay to every tool call of that kind, in every session.

#### Understanding

When a guard refuses something, you or the agent need to know why. With many overlapping mechanisms, the reason is harder to find, and fixes start to fight each other.

> Warning: put removal on a schedule, the same as fixes, or the harness only grows.

## Pruning on the same loop

Pruning uses the same steps as the improvement loop, pointed at the harness itself.

![Sequence diagram of pruning two guards that cover the same failure. The agent reads a month of guard fires and blocks, finds that Hook B blocked correct work and that Rule A covers the same failure, proposes merging both into one merge gate, and after your approval runs the task set before and after; with the same pass rate and fewer false blocks, it removes Rule A and replaces Hook B.]({{ '/assets/images/17-building-a-harness/pruning-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. Pruning two overlapping guards with a before-and-after run. The numbers in the diagram are illustrative. Diagram made with archify.*

### 1. Inventory

List every mechanism: each always-loaded context line, each skill, each hook, each script, each gate. For each, write the failure it exists to prevent. Two entries with the same failure are a merge candidate.

### 2. Measure

Measure each mechanism with the questions in [Measuring a guard's cost](#measuring-a-guards-cost) below.

### 3. Decide

| Finding | Decision |
|---|---|
| Fires often, blocks only bad work, cheap | Keep |
| Covers the same failure as another mechanism | Merge into the stronger one |
| Fires on correct work | Narrow its match, or move it later (from a hook to a merge gate) |
| Never fires, and the failure no longer happens | Retire it, then re-test |
| Written rule that agents break | Move it down the [enforcement ladder]({{ '/sections/17-building-a-harness/' | relative_url }}#3-prefer-a-fix-that-a-program-enforces) |

### 4. Re-test

Run your task set before and after the change. Keep the pruning if the pass rate holds. A rule that "never fires" may be preventing the failure by its presence, and the re-test is how you find out.

## Measuring a guard's cost

### The five questions

1. How often does it fire?
2. How often does it fire on work that was correct (a false block)?
3. How long does it take to run?
4. How much context does it add, counting both its always-loaded text and its refusal messages?
5. When it refuses, does the next attempt succeed?

### A worked example

An illustrative month of logs for three guards:

| Guard | Fires | False blocks | Time per run | Next attempt succeeds | Verdict |
|---|---|---|---|---|---|
| Pre-push secret scan | 40 | 0 | 1 s | Yes | Keep |
| "Run the tests" line in context | n/a | n/a | n/a | Agents still skip tests | Replace with a Stop hook |
| Pre-commit "no TODO" check | 25 | 9 | 2 s | Agents delete the comment and keep the problem | Retire; the review covers it |

The third row shows a guard with a net cost: it fires on correct work, and when it fires, agents delete the comment and leave the problem in place.

<details markdown="1">
<summary>How to get the numbers</summary>

1. Every hook appends one line per run to a log.
   - The time, the hook name, the decision and the duration.
2. An agent samples refusals from session transcripts.
   - For each one, it records whether the refused action was correct.
3. The agent counts fires, false blocks and run time per guard, and writes the table above.

```text
2026-11-03T14:02:11Z  pre-push-secret-scan  allow   0.9s
2026-11-03T14:20:45Z  no-todo-check         block   1.8s  file=src/parse.py
2026-11-03T14:21:02Z  no-todo-check         allow   1.7s
```

</details>

## Design habits that slow the creep

### Prefer one strong check over many weak ones

A single gate that checks the result often replaces several instructions that try to prevent the problem. A test of the output catches the failure whichever step caused it.

### Keep reusable parts apart from personal settings

Put the parts others could use (a skill, a hook, a template) in their own files with their own tests, and keep your personal preferences in a separate layer. Each part can then be understood, tested and removed on its own.

### Retire with the model

When the model changes, re-test the rules that compensated for the old one. See [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}).

## Examples from my own harness

My harness has gone through this: rules written for the same failure at different times, guards whose match was too broad, and context lines that compensated for an older model. The fixes were the ones on this page: merge overlapping rules, narrow or move noisy guards, retire rules after a re-test, and keep the context file short by having it point at the guards rather than restate them.

To start: list every always-loaded line in your context files and, next to each, the failure it prevents. Any line without an answer is a candidate for removal.

## Sources

- From the author's own harness, described conceptually.

{% include page_nav.html %}
