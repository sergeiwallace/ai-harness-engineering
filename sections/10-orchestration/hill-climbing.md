---
title: "Hill-climbing and quality ratchets"
parent: "Goals and loops"
grand_parent: "Orchestration and workflows"
nav_order: 1
---

{% include page_nav.html %}

# Hill-climbing and quality ratchets

As of 2026-11-09.

A sub-topic of [goals and loops]({{ '/sections/10-orchestration/goals-and-loops.html' | relative_url }}). Hill-climbing is a loop with a measurement in it: run, measure, change one thing, and keep the change only if the measure improved. A quality ratchet applies the same rule to a whole codebase over time: a number may improve, and a change that makes it worse is refused. This page covers the loop, local maxima, choosing a measure, running a climb in Claude Code, and ratchets.

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

## Running a climb in Claude Code

Two ways to run the loop without driving each round yourself:

| Way | How | Bound |
|---|---|---|
| A `/goal` | `/goal tests/test_classify.py pass rate printed after each change reaches 46/50, keeping only changes that raise it; or stop after 8 turns` | The turn clause |
| A dynamic workflow | "use a workflow to run the benchmark, change one thing, keep it only if the score rises, and stop when two rounds in a row make no progress" | The no-progress rule, and the 1,000-agent run cap |

The second phrasing follows an example prompt in the Claude Code workflows docs ("keep fixing the reported errors until the type check passes or two rounds in a row make no progress").

#### Example: one round, step by step

1. Read the baseline from the last kept commit.
2. Make one change and commit it on a branch.
3. Run the benchmark and print the score.
4. Compare.
   - Better and nothing else broke: keep the commit; it is the new baseline.
   - Not better: `git revert` the commit.
5. Record the round in a results table, so the history survives a compaction.

## Quality ratchets

A ratchet records the current value of a quality number (lint warnings, type errors, test coverage, complexity) and refuses any change that makes it worse. Improvements lower the recorded value, so the bar moves only one way.

`quality-baseline.json`, committed with the code and lowered automatically when a change improves a number:

```json
{ "lint_warnings": 212, "type_errors": 37, "functions_over_complexity_15": 9 }
```

A check on every commit compares the new counts with the file:

```text
lint_warnings: 212 -> 209   improved, baseline lowered
type_errors:    37 -> 38    REFUSED: one new type error in export.py:88
```

### Ratchet or target

| | Target ("zero warnings by June") | Ratchet |
|---|---|---|
| Asks of each change | Nothing until the deadline | Leave the number no worse |
| Failure mode | All or nothing; gets postponed | A legitimate regression needs an explicit exception |
| Progress | In one push near the deadline | A little with every improving change |
| Enforced by | A person remembering | A check on every commit |

### Where the ratchet runs

| Layer | Where | What it catches |
|---|---|---|
| Pre-commit hook | On the developer's or agent's machine | A regression before it is committed |
| CI job | On every pull request | A regression from any client, including one that skipped the hook |

A minimal ratchet for one number:

```python
#!/usr/bin/env python3
"""Fail if lint warnings rose; lower the baseline when they fell."""
import json
import pathlib
import subprocess
import sys

BASE = pathlib.Path("quality-baseline.json")
out = subprocess.run(["ruff", "check", ".", "--output-format", "json"],
                     capture_output=True, text=True)
count = len(json.loads(out.stdout or "[]"))
data = json.loads(BASE.read_text())
base = data["lint_warnings"]
if count > base:
    sys.exit(f"lint warnings rose from {base} to {count}: fix the new ones")
if count < base:
    data["lint_warnings"] = count
    BASE.write_text(json.dumps(data) + "\n")
    print(f"lint warnings fell from {base} to {count}: baseline lowered")
```

The script does the enforcing; a context-file line asking agents to "keep warnings down" is the second layer.

## Examples from my own harness

- A quality ratchet that lets code-quality numbers improve and refuses a change that makes them worse.
  - Agent-written changes run the same check as mine.
  - An agent cannot trade a passing feature for a new pile of warnings.

To start: record one number your linter prints today, and fail any change that raises it.

## Pitfalls and lessons

| Pitfall | What helps |
|---|---|
| Two changes in one round | One change per round, so the result has one cause |
| A measure the agent can game | Pair it with the full test suite or a held-out set |
| A climb that stalls on a local maximum | One larger change, or a restart from another baseline |
| A loop with no stop | A round cap, a target, or a no-progress rule |
| A ratchet with no exception path | Reviewed, counted exceptions in the code |

## Sources

- Workflow prompt pattern ("until ... two rounds in a row make no progress"): <https://code.claude.com/docs/en/workflows.md>, verified 2026-11-09.
- `/goal` turn clauses: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-09.
- Local maximum image: <https://commons.wikimedia.org/wiki/File:Local_maximum.png>, public domain, fetched 2026-11-09.

{% include page_nav.html %}
