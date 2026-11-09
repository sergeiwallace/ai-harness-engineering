---
title: "Hill-climbing and quality ratchets"
parent: "Goals and loops"
grand_parent: "Orchestration and workflows"
nav_order: 1
---

{% include page_nav.html %}

# Hill-climbing and quality ratchets

As of 2026-11-09.

A sub-topic of [goals and loops]({{ '/sections/10-orchestration/goals-and-loops.html' | relative_url }}). Hill-climbing is a loop with a measurement in it: run, measure, change one thing, and keep the change only if the measure improved. A quality ratchet applies the same rule to a whole codebase over time: a number may get better, never worse.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The loop

![A sequence diagram. The agent measures a baseline on the benchmark and gets 38 of 50. Inside a loop fragment it commits a change, measures again, gets 42 of 50 and keeps the change; then commits another change, measures 41 of 50, and reverts it.]({{ '/assets/images/10-orchestration/hill-climb-rounds.svg' | relative_url }}){: style="width:100%"}

*Figure: hill-climbing keeps a change only when the measure improves. Made with archify.*

### The four rules

1. Measure before changing anything. Without a baseline there is no "better".
2. Change one thing per round, so a result has one cause.
3. Keep a change only if the measure improved and nothing else broke.
4. Stop at a limit: a round count, a target, or a run of rounds with no gain.

## Local maxima

![A surface with two peaks: a lower peak near the origin and a higher peak further along. A climb that starts near the lower peak and only accepts improvements stops on top of it.]({{ '/assets/images/10-orchestration/local-maximum.png' | relative_url }}){: style="width:100%"}

*Figure: two peaks on one surface. Image by Headlessplatter, released into the public domain. Source: <https://commons.wikimedia.org/wiki/File:Local_maximum.png>.*

Hill-climbing accepts only changes that improve the measure, one small step at a time. Starting near the lower peak, every small step away from its top makes the measure worse, so the loop stops there even though a higher peak exists.

For agent work this shows up as a run of rounds with no gain, as in the prompt example below. Two ways to respond:

1. Try a larger change for one round.
   - For a prompt: a different structure, such as asking for the reason before the label, in place of another wording tweak.
2. Restart from a different baseline and climb again, then keep the better of the two results.

## What makes a good measure

| Good measure | Why | Weak measure | Why |
|---|---|---|---|
| Test suite wall time | Objective, cheap to rerun | "Code looks cleaner" | No number |
| Pass rate of a prompt on 50 fixed examples | Fixed set, comparable | Pass rate on new examples each round | The set changes, not the prompt |
| Count of lint warnings | Exact | Lines of code | Fewer lines can be worse code |

> An agent can raise a measure without improving the behaviour behind it, for example by special-casing the benchmark's examples. Pair every measure with a check that the real behaviour still works, such as the full test suite.
{: .warning }

## Worked example: a prompt's pass rate

A classification prompt is tested on 50 fixed examples with known answers.

| Round | Change | Pass rate | Kept? |
|---|---|---|---|
| 0 | Baseline | 38/50 | |
| 1 | Add two labelled examples to the prompt | 42/50 | Yes |
| 2 | Reword the category definitions | 41/50 | No, reverted |
| 3 | Ask for the reason before the label | 45/50 | Yes |
| 4 | Lower temperature | 45/50 | No gain, reverted |
| 5 | Split one ambiguous category in two | 44/50 | No, reverted |

After three rounds with no gain, the loop stops at 45/50 and reports the five remaining failures for a person to look at. They turn out to be examples with disputed answers.

## Quality ratchets

A ratchet records the current value of a quality number (lint warnings, type errors, test coverage, complexity) and refuses any change that makes it worse. Improvements lower the recorded value, so the bar moves only one way.

```yaml
# quality-baseline.yml: lowered automatically when a change improves it
lint_warnings: 212
type_errors: 37
functions_over_complexity_15: 9
```

A check on every commit compares the new counts with the file:

```text
lint_warnings: 212 -> 209   improved, baseline lowered
type_errors:    37 -> 38    REFUSED: one new type error in export.py:88
```

### Why a ratchet beats a target

A target ("zero warnings by June") is all or nothing and gets postponed. A ratchet asks only that each change leaves things no worse, which every change can do. Each improvement becomes the new recorded value.

<details markdown="1">
<summary>What to do about a legitimate regression</summary>

Sometimes a change has to add a warning, for example a deliberate broad exception at a process boundary. Allow an explicit, reviewed exception with a reason in the code, and count exceptions separately so they stay visible.

</details>

## Examples from my own harness

- A quality ratchet that lets code-quality numbers improve and refuses a change that makes them worse.
  - Agent-written changes run the same check as mine.
  - An agent cannot trade a passing feature for a new pile of warnings.

To start: record one number your linter prints today, and fail any change that raises it.

{% include page_nav.html %}
