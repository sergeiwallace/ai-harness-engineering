---
title: "Detecting test edits in a diff"
parent: "Test tampering and oracle integrity"
grand_parent: "Testing agent-written code"
nav_order: 1
---

{% include page_nav.html %}

# Detecting test edits in a diff
{: .no_toc }

As of 2026-11-09. Part of [Test tampering and oracle integrity]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}), in [Testing agent-written code]({{ '/sections/12-testing/' | relative_url }}).

One low-cost check against [test tampering]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}) is a check that lists every pre-existing test a branch changed. Both 2026 reward-hacking benchmarks used this as a detection channel: one measured hacking partly through "test file edit detection", and the other flagged any run in which the test files were modified. It costs one `git` command and no model call. This page shows how to build it, what to do with its output, and what it cannot see.

<figure>
  <img src="{{ '/assets/images/12-testing/test-edit-flag.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: launcher, flag tool, branch history and coding agent. The launcher runs the flag tool against main. The tool asks the branch history for modified or deleted test files and gets three; it reads the Test-edit-reason trailers and finds two carry a reason. It reports one changed test with no reason, listing the removed assertion lines. The launcher reads the hunk against the criteria and sends the agent a finding to restore or justify; the agent restores it and re-runs green.">
  <figcaption>A launcher checking a delegated branch before merge. Made with archify.</figcaption>
</figure>

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## The core command

### List changed and deleted test files

A test file the branch added is new work and is reviewed as such. The question here is narrower: which tests that existed before the branch did it change or delete?

```bash
base=$(git merge-base origin/main HEAD)
git diff --name-only --diff-filter=MD "$base"...HEAD -- \
  'tests/' '*_test.py' 'test_*.py' '*.test.ts' '*.spec.ts'
```

`--diff-filter=MD` keeps only modified (`M`) and deleted (`D`) paths, so newly added tests drop out. The three-dot range compares the branch tip with the point it forked from, so changes that landed on main in the meantime do not show up.

### Show the removed assertions

The lines that matter most are assertions the branch removed:

```bash
git diff "$base"...HEAD -- tests/ | grep -E '^-\s*(assert|expect|self\.assert)'
```

A removed `assert` with a replacement nearby is often loosening; a removed `assert` with nothing in its place is deletion.

### Look for new skips

```bash
git diff "$base"...HEAD -- tests/ | grep -E '^\+.*(pytest\.mark\.(skip|xfail)|\.skip\(|it\.only|xit\()'
```

## Pairing the check with stated reasons

Tests also change for good reasons, so a bare list of changed tests includes legitimate edits. Asking the agent to state each reason in a form a script can read separates the two.

### A commit trailer

```text
fix(export): omit archived items from the default export

Test-edit-reason: AC3 changed (archived items omitted from the default export)
```

Git reads trailers natively:

```bash
git log --format='%H %(trailers:key=Test-edit-reason,valueonly)' "$base"..HEAD
```

### The decision table

| Test changed? | Reason trailer? | What the launcher does |
|:--|:--|:--|
| No | n/a | Nothing |
| Yes | Yes, cites a changed criterion | Check the criterion really changed; usually accept |
| Yes | Yes, "moved to its real boundary" | Check the new test still asserts the behaviour |
| Yes | No | A finding: restore the test or justify the change |

### Worked example: the report

```text
Pre-existing tests changed on this branch (base 3f2a9c1):
  tests/test_export.py        reason: AC3 changed (archived items omitted ...)
  tests/test_cli_priority.py  reason: moved to entry point (was testing parse_priority)
  tests/test_sync.py          NO REASON
    - assert result.conflicts == []
    + assert result is not None
1 unreasoned test edit. Read it against the acceptance criteria before merge.
```

In the third entry the assertion went from "no conflicts" to "returned something", which passes for any result.

## What it cannot see

| Gap | Example | Covered by |
|:--|:--|:--|
| Special-casing in the code | `if path == FIXTURE_PATH: return EXPECTED` | Review of the code diff; held-out tests |
| A weak test that was always weak | A test with no assertion at all | Mutation testing on core logic |
| Edits through a renamed file | The test is moved and loosened in one step | Add `--find-renames` and treat `R` as `M` |
| Fixture changes | The expected data file changed, not the test | Include fixture paths in the path list |

## From my harness

The launcher in my setup runs a small tool of this shape over every delegated branch before merge. It lists modified and deleted pre-existing tests, matches each against the branch's `Test-edit-reason:` trailers, and prints the unmatched ones with their removed lines. An unmatched edit is a review finding. It does not block the merge on its own, because a block would also stop legitimate edits.

## Sources

- Escalation channels study (test-file modification flag): <https://arxiv.org/abs/2608.29460>, verified 2026-11-09.
- EvilGenie (test file edit detection): <https://arxiv.org/abs/2511.21654>, verified 2026-10-08.
- Git trailers: <https://git-scm.com/docs/git-interpret-trailers>, verified 2026-11-09.

{% include page_nav.html %}
