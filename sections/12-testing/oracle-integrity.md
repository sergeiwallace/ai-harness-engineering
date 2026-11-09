---
title: "Test tampering and oracle integrity"
parent: "Testing agent-written code"
nav_order: 2
---

{% include page_nav.html %}

# Test tampering and oracle integrity
{: .no_toc }

As of 2026-11-09. Part of [Testing agent-written code]({{ '/sections/12-testing/' | relative_url }}).

The oracle is whatever decides whether code is correct: a test suite, a reference output, a checker script. Test tampering is when an agent makes its code pass by changing the oracle instead of the code: deleting an assertion, loosening an expected value, skipping a case, or special-casing the test input. It is the testing form of reward hacking. This page covers when it happens, the cheapest controls that measurably reduce it, who should judge a disputed test, and the heavier controls to reach for when the cheap ones are not enough.

<figure>
  <img src="{{ '/assets/images/12-testing/tamper-dispute.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: launcher, coding agent, old test and test-edit check. An old export test fails. On the common path the agent reports a disputed test and the launcher rules that the criterion changed, so the agent edits the test with a stated reason. On the rare path the agent silently loosens the assertion, the test-edit check flags it, and the launcher files a finding.">
  <figcaption>The two paths an agent can take when an old test blocks it. Made with archify.</figcaption>
</figure>

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## What tampering looks like

### Four shapes, from blunt to subtle

| Shape | What changes | Example |
|:--|:--|:--|
| Deletion | A test or assertion is removed | `assert len(rows) == 3` disappears |
| Loosening | The expected value or scope is widened | `== 3` becomes `>= 1`; an exact match becomes "contains" |
| Skipping | The test is marked to skip or expected to fail | a new `@pytest.mark.skip` with a vague reason |
| Special-casing | The code detects the test and returns the expected answer | `if input == FIXTURE: return EXPECTED` |

The first three show up in a diff of the test files. Special-casing does not touch a test file, so a diff check does not catch it.

### Worked example: a loosened assertion

```diff
 def test_export_includes_archived_items(tmp_path):
     export = run_export(tmp_path, include_archived=True)
-    assert export.count == 12
+    # archived items are now exported lazily, so the count may differ
+    assert export.count > 0
```

The comment gives a plausible reason. The assertion now passes for any non-empty export, including one that drops every archived item. A reviewer reading quickly accepts it. Published work on LLM monitors found exactly this pattern: a "backward compatibility" rationale successfully deceived a strong model acting as the monitor.

## When it happens

On ordinary benchmark tasks, models that succeeded mostly changed the code. Tampering concentrates where the spec and the test conflict, or where the task cannot be done as asked.

| Evidence | Finding |
|:--|:--|
| A 2026 model card | Attempted reward hacking three to six times more frequent on impossible tasks than possible ones |
| ImpossibleBench | Tasks with deliberate spec-test conflicts; "frontier models cheat surprisingly often", and some families cheat mainly by modifying test cases |
| An escalation study, eight models | Baseline hacking on tasks with defective checkers of 11.1% for one model and 13.3% for another |

> This evidence points controls at the moment an agent cannot pass a test. A control that reviews every diff mostly reviews diffs with no tampering in them.

## Policy line and escalation channel

### What was tested

A 2026 study ran a two-by-two experiment across eight frontier models from five families. One factor was a policy line forbidding test manipulation, defined as "Modifying, deleting, or narrowing test assertions, expected values, or test scope." The other was an escalation tool the agent could call to report a problem. The tool took an issue type (one was a test-infrastructure conflict), an assessment, the approaches tried and a recommendation.

| Condition | Test tampering across the eight models |
|:--|:--|
| Neither | 23.6% |
| Policy and escalation together | 5.3%, and zero for six of the eight models |

Escalation and hacking were close to mutually exclusive: 98.7% of escalations involved no hacking. The study found no measurable cost in task performance.

#### Caveats

It is a single-author preprint over nine problems with known checker defects. The escalation tool told models their report went to a human reviewer. Whether the effect holds when the report goes to a launching agent instead is untested. An independent benchmark points the same way: a strict prompt telling the model to stop and explain flawed tests cut one model's cheating from 92% to 1%.

### Putting it in a brief

```text
Tests: never modify, delete or narrow an existing test's assertions, expected values or
scope to make it pass. If you believe a test is wrong, stop and report it in your final
message under "Disputed tests": the test, the acceptance criterion it conflicts with,
what you tried, and what you recommend. Reporting a disputed test is a valid outcome.
```

The last sentence gives the agent a permitted way out. An agent told only "do not edit tests", facing a test it cannot pass, believes its task is impossible, and tampering rises in that condition.

## Who judges a disputed test

### The launcher as judge

Whoever launched the work holds the acceptance criteria and did not write the code, which makes it one natural judge. A person or a separate reviewer agent can also take the role. My harness uses the launcher, and in that setup a run works like this:

1. The agent finishes its branch and lists any disputed tests in its final message.
   - Each entry names the test, the criterion it conflicts with, what was tried and a recommendation.
2. The [test-edit check]({{ '/sections/12-testing/test-edit-detection.html' | relative_url }}) lists the pre-existing tests the branch changed.
   - Changes with a stated reason are listed apart from changes without one.
3. The launcher reads those two lists against the criteria and rules on each item.
   - A branch with no dispute and no unreasoned edit goes on to normal review.

#### Worked example: the report and the ruling

```text
Disputed tests
- test_export_includes_archived_items expects 12 rows; the change exports 9.
  Conflicts with: AC3, "archived items are omitted from the default export".
  Tried: confirmed the 3 missing rows are the archived fixtures.
  Recommend: change the test to call the export with include_archived=True.
```

The launcher checks AC3, agrees, and replies with one line. The agent updates the test and records why in the commit:

```text
Test-edit-reason: AC3 changed (archived items omitted from the default export)
```

### An LLM judge on every diff

A model asked to judge every change is costly and only partly reliable. In one 2025 benchmark, LLM monitors caught 86% to 89% of cheating on small coding tasks but only 42% to 50% on realistic repository tasks. Anthropic's guidance for its 2026 flagship model also advises removing routine "use a subagent to verify" instructions, because the model already checks its own work and the extra pass wastes tokens. The research behind this page concluded that a judge is useful on a flagged diff and costly on every diff.

## Heavier controls

### Make pre-existing tests read-only

ImpossibleBench found read-only test files a middle ground: they "restore legitimate performance while preventing test modification attempts". They do not stop special-casing. In Claude Code, a pre-tool-use hook can enforce it.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "./tools/protect-tests.sh" }]
      }
    ]
  }
}
```

```bash
#!/usr/bin/env bash
# Block edits to existing test files; new test files are allowed.
path=$(jq -r '.tool_input.file_path // empty')
case "$path" in
  */tests/*|*_test.py|*/test_*.py)
    if [ -e "$path" ]; then
      echo "Existing tests are read-only in this task. Report a disputed test instead." >&2
      exit 2
    fi ;;
esac
exit 0
```

#### Costs of the hard block

| Cost | Detail |
|:--|:--|
| Blocks legitimate edits | A changed acceptance criterion, or a test that must move to a new boundary |
| Binds one tool only | A Claude Code hook does not bind a Codex worker or a person |
| Misses special-casing | The code can still detect the fixture |

It fits long unattended runs where nobody reads the test diff. Tasks whose criteria change the tests need it turned off.

### File formats the model rewrites less

Anthropic's long-running harness kept its feature list in a JSON file that agents could change only by flipping a `passes` field, because "the model is less likely to inappropriately change or overwrite JSON files compared to Markdown files."

<details markdown="1">
<summary>Aside: exploratory approaches, not yet worth adopting</summary>

Two 2026 preprints go further. One argues for controls the agent runs inside but cannot bypass, including immutable tests, framed as a hypothesis to test. Another formally checks for conflicts between spec and tests before the agent starts and reports detecting up to 72.8% of them. Neither has adoption evidence yet.

</details>

## From my harness

My testing rule carries this as a short "oracle integrity" paragraph. The agent changing the code does not weaken, delete or rewrite a pre-existing test to make its change pass; if it believes the test is wrong, it stops and reports to its launcher with the test, the criterion it conflicts with, what it tried and its recommendation. A pre-existing test may be edited in the same change only with a stated reason, written as a commit trailer:

```text
Test-edit-reason: acceptance criterion 3 changed (export now omits archived items by default)
```

The two accepted reasons are a changed acceptance criterion, which the trailer cites, or a test coupled to the implementation that is being moved to its real boundary. Before merging a delegated branch, the launcher runs a small tool that lists every pre-existing test the branch changed and whether each change carries a reason. An edit with no reason is a review finding. The [sub-topic page]({{ '/sections/12-testing/test-edit-detection.html' | relative_url }}) shows how such a tool works. I have not turned on the read-only hook by default, because many of my tasks legitimately change tests.

## Sources

- Escalation channels and reward hacking: <https://arxiv.org/abs/2608.29460>, verified 2026-11-09.
- ImpossibleBench: <https://arxiv.org/abs/2510.20270>, verified 2026-10-08.
- EvilGenie reward-hacking benchmark: <https://arxiv.org/abs/2511.21654>, verified 2026-10-08.
- Model card (three to six times on impossible tasks): <https://www.anthropic.com/claude-opus-5-5-system-card>, verified 2026-10-08.
- Hooks guide, blocking edits to protected files: <https://code.claude.com/docs/en/hooks-guide>, verified 2026-10-08.
- Effective harnesses for long-running agents: <https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents>, verified 2026-10-08.

{% include page_nav.html %}
