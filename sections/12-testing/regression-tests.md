---
title: "The bug-fix regression test"
parent: "Testing agent-written code"
nav_order: 3
redirect_from:
  - /sections/12-testing/advanced.html
---

{% include page_nav.html %}

# The bug-fix regression test
{: .no_toc }

As of 2026-11-09. Part of [Testing agent-written code]({{ '/sections/12-testing/' | relative_url }}).

A bug fix is the one place where the red step of test-driven development still pays for itself with agents. The regression test must fail on the unfixed code, for the reason the bug report describes, before it may pass on the fix. This page walks through the full method: reproduce, find the root cause, freeze the test, fix, verify in widening rings, and decide where each rule is enforced.

<figure>
  <img src="{{ '/assets/images/12-testing/regression-proof.svg' | relative_url }}" width="100%" alt="Sequence diagram with five lanes: coding agent, unfixed code, regression test, fixed code and full suite. The agent reproduces the bug, writes a test at the entry point, and sees it fail on the unfixed code for the stated reason. It commits the failing test alone. It writes the fix, the same test passes on the fixed code, and the preservation tests stay green.">
  <figcaption>Red then green, with the test frozen in between. Made with archify.</figcaption>
</figure>

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## The red step in bug fixes

For new features, strict TDD inside the agent loop showed no measurable gain in one 2026 evaluation (see [test ordering and the spec]({{ '/sections/12-testing/entry-point-testing.html#test-ordering-and-the-spec' | relative_url }})). A bug report comes with a concrete observed failure, so there is a precise thing the test must reproduce, and a precise way to check the test does its job: it fails on the old code.

The open-source test-audit skill says: "Bug regression tests must fail on the pre-fix code for the intended reason." Anthropic's example prompt for a bug is the same order: "write a failing test that reproduces the issue, then fix it."

> The agent-specific risk is a test written after the fix, which passes on both the old and new code. It has the form of a regression test and does not detect the bug.

## The method, step by step

### 1. Reproduce

Drive the reported input through the real entry point and observe the reported wrong output. A fix for a bug that was never reproduced cannot be checked against the reported failure.

#### Worked example

A user reports that `task export --format csv` drops tasks whose title contains a comma.

```console
$ task new 'Fix login, then deploy'
created 42
$ task export --format csv | grep -c '^42,'
0
```

### 2. Find the root cause

Trace the failure back until it points at a line of code. Here the CSV writer joins fields with `,` without quoting. The root cause is "fields are not quoted", not "commas break export". The root cause also breaks titles with quotes and newlines, so the fix should cover those too.

### 3. Freeze the failing test

Write the test at the level the defect lives at, run it on the unfixed code, and confirm it fails for the stated reason.

```python
def test_csv_export_keeps_titles_with_commas(tmp_path):
    task_id = run_cli(["new", "Fix login, then deploy"], home=tmp_path).task_id
    out = run_cli(["export", "--format", "csv"], home=tmp_path).stdout
    rows = list(csv.reader(io.StringIO(out)))
    assert [task_id, "Fix login, then deploy"] in [r[:2] for r in rows]
```

```console
$ pytest tests/test_export.py::test_csv_export_keeps_titles_with_commas
FAILED  AssertionError: ['42', 'Fix login, then deploy'] not in [['42', 'Fix login', ' then deploy'], ...]
```

The failure message shows the title split in two, which is the bug as reported. A failure for any other reason (an import error, a missing fixture) does not count.

Commit the test on its own. From here on, the fix commit may not touch this file.

### 4. Fix

Make the smallest change that addresses the root cause, here using a CSV library's quoting instead of a hand-written join.

### 5. Verify in widening rings

| Ring | What runs | What it proves |
|:--|:--|:--|
| 1 | The frozen regression test | The reported bug is fixed |
| 2 | Tests for the touched module | Nearby behaviour is preserved |
| 3 | Tests the diff's impact map selects | Callers are preserved |
| 4 | The full suite on the merged result | Nothing else broke |
| 5 | The live system, where one exists | The fix holds outside the test environment |

Each ring is slower and runs less often. Stopping at ring 1 is the common agent shortcut.

<details markdown="1">
<summary>Aside: the five-line proof a reviewer should ask for</summary>

A credible bug-fix report records: the reproduction fails on the clean base; the same reproduction passes with the fix; preservation tests pass; the fix did not alter the frozen test or the gate; and the result was re-run from a clean checkout. Do not accept an agent's statement that a failure is "pre-existing". Run the test on the base commit and see.

</details>

## Where each rule is enforced

A rule written in a procedure is the weakest form of enforcement. An agent can skip it without anything noticing. Rank each rule by the strongest mechanism that can carry it.

| Rule | Procedure | Hook | CI or gate |
|:--|:--|:--|:--|
| Reproduce before fixing | Yes | No | No |
| The test fails on the base | Yes | No | Yes: run the new test on the merge base |
| The fix does not touch the frozen test | Yes | Possible | Yes: diff check on the test file |
| Widening rings | Yes | No | Partly: the gate selects tests from the diff |

The order of preference is CI or gate first, then hook, then written procedure. [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers this ladder in general.

## From my harness

My bug-fix procedure is a skill with numbered gates, in this order: reproduce, root-cause, freeze the failing regression test, fix, verify, document. Its freeze gate requires that the regression test "must assert the real behavioral contract", at whatever level the defect actually lives (unit, integration, end-to-end or front end), and that the test is not edited during the fix.

## Sources

- OpenClaw test-audit skill: <https://github.com/openclaw/openclaw/blob/main/.agents/skills/test-audit/SKILL.md>, verified 2026-10-08.
- Claude Code best practices: <https://code.claude.com/docs/en/best-practices>, verified 2026-10-08.
- TDD inside the agent loop: <https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html>, verified 2026-10-08.

{% include page_nav.html %}
