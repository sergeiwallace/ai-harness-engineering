---
title: "The bug-fix regression test"
parent: "Testing agent-written code"
nav_order: 3
redirect_from:
  - /sections/12-testing/advanced.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# The bug-fix regression test
{: .no_toc }

A bug fix is the one place where the red step of test-driven development still pays for itself with agents. The regression test must fail on the unfixed code, for the reason the bug report describes, before it may pass on the fix.

This page walks one bug through the full method:

1. reproduce;
2. find the root cause;
3. freeze the failing test;
4. fix;
5. verify in widening rings.

It then covers where each rule is enforced, and the pitfalls.

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

For new features, strict TDD inside the agent loop showed no measurable gain in one 2026 evaluation (see [test ordering and the spec]({{ '/sections/12-testing/entry-point-testing.html#test-ordering-and-the-spec' | relative_url }})). A bug fix is different in two ways:

- A bug report comes with a concrete observed failure, so there is a precise behaviour for the test to reproduce.
- Running the test on the old code shows whether the test does its job: it must fail there.

Two sources state the same order:

| Source | Wording |
|:--|:--|
| OpenClaw's test-audit skill | "Bug regression tests must fail on the pre-fix code for the intended reason." |
| Anthropic's Claude Code best practices | "write a failing test that reproduces the issue, then fix it" |

## The method, step by step

### 1. Reproduce

Drive the reported input through the real entry point and observe the reported wrong output. A fix for a bug that was never reproduced cannot be checked against the reported failure.

#### Worked example

A user reports that after `task export --format csv`, a spreadsheet shows titles containing a comma split across two columns, with the status in the wrong column. Reproduce it with the same CSV parsing the spreadsheet does:

```console
$ task new 'Fix login, then deploy'
created 42
$ task export --format csv | python3 -c 'import csv, sys; print(list(csv.reader(sys.stdin))[1])'
['42', 'Fix login', ' then deploy', 'open']
```

The row has four fields where three are expected, which is the reported failure.

### 2. Find the root cause

Trace the failure back until it points at a line of code:

1. Symptom: titles with a comma split into two columns.
2. Where it happens: the exported line for task 42 is `42,Fix login, then deploy,open`.
3. Cause: the CSV writer joins fields with `,` and never quotes them.

```python
line = ",".join([task.id, task.title, task.status])   # no quoting
```

The cause, unquoted fields, breaks more than commas:

| Title | Result before the fix |
|:--|:--|
| `Fix login, then deploy` | Split into two fields |
| `Say "hi"` | The quote characters reach the parser unescaped |
| A title with a line break | One task becomes two rows |

### 3. Freeze the failing test

Write the test at the level the defect lives at, run it on the unfixed code, and confirm it fails for the stated reason.

```python
def test_csv_export_keeps_titles_with_commas(tmp_path):
    task_id = run_cli(["new", "Fix login, then deploy"], home=tmp_path).task_id
    out = run_cli(["export", "--format", "csv"], home=tmp_path).stdout
    rows = list(csv.reader(io.StringIO(out)))
    assert [task_id, "Fix login, then deploy", "open"] in rows
```

```console
$ pytest tests/test_export.py::test_csv_export_keeps_titles_with_commas
FAILED  AssertionError: ['42', 'Fix login, then deploy', 'open'] not in [['id', 'title', 'status'], ['42', 'Fix login', ' then deploy', 'open']]
```

Check the failure reason:

- The message shows the title split in two, which is the bug as reported.
- A failure for any other reason (an import error, a missing fixture) does not count; fix the test until it fails for the right reason.

Commit the test on its own. From here on, the fix commit may not touch this file.

### 4. Fix

Make the smallest change that addresses the root cause. Here that is the standard library's CSV writer, which quotes fields as needed:

```python
writer = csv.writer(out)
writer.writerow([task.id, task.title, task.status])
```

Then cover the rest of the root cause:

1. Run the frozen test on the fix; it passes.
2. Add tests for the quote and line-break titles in a separate commit.
3. Run those new tests on the base commit too, and confirm they fail there.
   - A new case that already passed on the old code is not a regression test for this bug.

### 5. Verify in widening rings

| Ring | What runs | What it proves |
|:--|:--|:--|
| 1 | The frozen regression test | The reported bug is fixed |
| 2 | Tests for the touched module | Nearby behaviour is preserved |
| 3 | Tests the diff's impact map selects | Callers are preserved |
| 4 | The full suite on the merged result | Nothing else broke |
| 5 | The live system, where one exists | The fix holds outside the test environment |

Each ring is slower and runs less often. Stopping at ring 1 is the common agent shortcut.

## Where each rule is enforced

A rule written only in a procedure is the weakest form of enforcement: an agent can skip it without anything noticing. Rank each rule by the strongest harness layer that can carry it (see [Where tests sit in the harness]({{ '/sections/12-testing/#where-tests-sit-in-the-harness' | relative_url }})):

| Rule | Layer 2, workflow | Layer 3, hook or script | Layer 4, gate |
|:--|:--|:--|:--|
| Reproduce before fixing | The procedure's first step | No | No |
| The test fails on the base | The procedure | No | Yes: run the new test on the merge base |
| The fix does not touch the frozen test | The procedure | Possible: a hook on the test path | Yes: diff check on the test file |
| Widening rings | The procedure | No | Partly: the gate selects tests from the diff |

The gate check for "fails on the base" fits in a few lines:

```bash
# Run the branch's new regression test against the merge-base code; it must FAIL there
base=$(git merge-base origin/main HEAD)
git worktree add --detach /tmp/base-check "$base"
cp tests/test_export.py /tmp/base-check/tests/
( cd /tmp/base-check && ! pytest -q tests/test_export.py::test_csv_export_keeps_titles_with_commas )
status=$?
git worktree remove --force /tmp/base-check
exit $status
```

The `!` inverts the result, so the gate passes only when the test fails on the old code. [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers the enforcement ladder in general.

## Pitfalls

| Pitfall | Example | Fix |
|:--|:--|:--|
| The test was written after the fix | It passes on old and new code | Run it on the base commit and see it fail |
| It fails for the wrong reason | An import error on the base commit | Read the failure message against the bug report |
| The fix edits the frozen test | The assertion is loosened to match the new output | A diff check on the test file in the fix commit |
| Only the reported case is fixed | Commas work, quotes and line breaks still break | Fix the root cause and add its sibling cases |
| "Pre-existing failure" accepted on the agent's word | A real regression is waved through | Run the failing test on the base commit |

## From my harness

My bug-fix procedure is a skill with numbered gates:

1. Reproduce.
2. Establish the root cause.
3. Freeze the regression test.
   - It "must assert the real behavioral contract", at whatever level the defect lives: unit, integration, end-to-end or front end.
   - It is confirmed failing on the unfixed code.
4. Apply the causal fix.
5. Verify in widening rings.
6. Document the bug.

The skill is layer 2 of the harness: it builds the order into the workflow the agent follows. The merge gate checks that the frozen test passes, which is layer 4.

## Sources

- [OpenClaw test-audit skill](https://github.com/openclaw/openclaw/blob/main/.agents/skills/test-audit/SKILL.md), verified 2026-10-08.
- [Claude Code best practices](https://code.claude.com/docs/en/best-practices), verified 2026-10-08.
- [TDD inside the agent loop](https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html), verified 2026-10-08.

{% include page_nav.html %}
