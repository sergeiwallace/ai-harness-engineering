---
title: "Detecting test edits in a diff"
parent: "Test tampering and oracle integrity"
grand_parent: "Testing agent-written code"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Detecting test edits in a diff
{: .no_toc }

One low-cost check against [test tampering]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}) lists every pre-existing test a branch changed. It costs a few `git` commands and no model call. Two 2026 reward-hacking benchmarks used it as a detection channel:

- EvilGenie measured hacking partly through "test file edit detection".
- The escalation-channel study flagged any run in which the test files were modified during execution.

This page builds the check step by step, pairs it with stated reasons, assembles it into one script, and lists what it cannot see.

<figure>
  <img src="{{ '/assets/images/12-testing/test-edit-flag.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: launcher, flag tool, branch history and coding agent. The launcher runs the flag tool against main. The tool asks the branch history for modified or deleted test files and gets three; it reads the Test-edit-reason trailers and finds two carry a reason. It reports one changed test with no reason, listing the removed assertion lines. The launcher reads the hunk against the criteria and sends the agent a finding to restore or justify; the agent restores it and re-runs green.">
  <figcaption>A launcher checking a delegated branch before merge. Made with archify.</figcaption>
</figure>

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## The core commands

### List changed and deleted test files

A test file the branch added is new work and is reviewed as such. The question here is narrower: which tests that existed before the branch did it change, rename or delete?

```bash
base=$(git merge-base origin/main HEAD)
git diff --name-only --find-renames --diff-filter=MDR "$base"...HEAD -- \
  'tests/' '*_test.py' 'test_*.py' '*.test.ts' '*.spec.ts'
```

What each part does:

- `--diff-filter=MDR` keeps modified (`M`), deleted (`D`) and renamed (`R`) paths, so newly added tests drop out.
- `--find-renames` reports a moved test as a rename, so a test moved and loosened in one step still shows up.
- The three-dot range `"$base"...HEAD` compares the branch tip with the point it forked from, so changes that landed on main in the meantime do not show up.

### Show the removed assertions

The lines that matter most are assertions the branch removed:

```bash
git diff "$base"...HEAD -- tests/ | grep -E '^-\s*(assert|expect|self\.assert)'
```

| Pattern in the diff | Usually means |
|:--|:--|
| A removed `assert` with a changed one next to it | Loosening; compare the two |
| A removed `assert` with nothing in its place | Deletion |
| A removed whole test function | Deletion; check the reason |

### Look for new skips

```bash
git diff "$base"...HEAD -- tests/ | grep -E '^\+.*(pytest\.mark\.(skip|xfail)|\.skip\(|it\.only|xit\()'
```

`it.only` is included because it silently skips every other test in the file.

## Pairing the check with stated reasons

Tests also change for good reasons, so a bare list of changed tests includes legitimate edits:

| Reason a test changed | Legitimate? |
|:--|:--|
| The acceptance criterion changed | Yes, cite the criterion |
| The test was coupled to the implementation and moves to its real boundary | Yes, if it still asserts the behaviour |
| The test was weakened so the change passes | No |

Asking the agent to state each reason in a form a script can read separates the first two rows from the third.

### A commit trailer

A trailer is a `Key: value` line in the block at the end of a commit message, the same mechanism as `Signed-off-by:`.

```text
fix(export): omit archived items from the default export

The default export now skips archived items, per AC3.

Test-edit-reason: AC3 changed (archived items omitted from the default export)
```

Git reads trailers natively:

```bash
git log --format='%H %(trailers:key=Test-edit-reason,valueonly)' "$base"..HEAD
```

Two rules decide whether git sees the trailer:

1. It must be in the trailer block at the end of the message, after a blank line.
   - Git's documentation: "The trailer block is by definition at the end of the commit message." A `Test-edit-reason:` line in the middle of the body is not a trailer.
2. Every trailer-adding step must keep it last.
   - A tool that appends its own block after the body, or a squash merge that rewrites the message, can push the reason out of the trailer block.

### The decision table

| Test changed? | Reason trailer? | What the launcher does |
|:--|:--|:--|
| No | n/a | Nothing |
| Yes | Yes, cites a changed criterion | Check the criterion really changed; usually accept |
| Yes | Yes, "moved to its real boundary" | Check the new test still asserts the behaviour |
| Yes | No | A finding: restore the test or justify the change |

## Putting it together

A minimal version of the whole check as one script:

```bash
#!/usr/bin/env bash
# test-edit-flag: list pre-existing tests a branch changed, and whether a reason was given
set -euo pipefail
base=$(git merge-base "${1:-origin/main}" HEAD)
changed=$(git diff --name-only --find-renames --diff-filter=MDR "$base"...HEAD -- \
  'tests/' '*_test.py' 'test_*.py' '*.test.ts' '*.spec.ts')
[ -z "$changed" ] && { echo "No pre-existing tests changed."; exit 0; }

unreasoned=0
for f in $changed; do
  # commits on the branch that touched this file
  commits=$(git log --format=%H "$base"..HEAD -- "$f")
  reason=$(for c in $commits; do
    git log -1 --format='%(trailers:key=Test-edit-reason,valueonly)' "$c"; done | grep -v '^$' | head -1 || true)
  if [ -n "$reason" ]; then
    printf '  %-28s reason: %s\n' "$f" "$reason"
  else
    printf '  %-28s NO REASON\n' "$f"
    git diff "$base"...HEAD -- "$f" | grep -E '^[-+]\s*(assert|expect)' | sed 's/^/    /' || true
    unreasoned=$((unreasoned + 1))
  fi
done
echo "$unreasoned unreasoned test edit(s)."
```

It exits 0 in every case: the output is a review input, and the launcher decides.

### Worked example: the report

```text
  tests/test_export.py         reason: AC3 changed (archived items omitted ...)
  tests/test_cli_priority.py   reason: moved to entry point (was testing parse_priority)
  tests/test_sync.py           NO REASON
    - assert result.conflicts == []
    + assert result is not None
1 unreasoned test edit(s).
```

In the third entry the assertion went from "no conflicts" to "returned something", which passes for any result. The launcher reads that hunk against the acceptance criteria and sends the agent a finding.

## What it cannot see

| Gap | Example | Covered by |
|:--|:--|:--|
| Special-casing in the code | `if path == FIXTURE_PATH: return EXPECTED` | Review of the code diff; held-out tests |
| A weak test that was always weak | A test with no assertion at all | [Mutation testing]({{ '/sections/12-testing/#measuring-test-strength' | relative_url }}) on core logic |
| Fixture changes | The expected data file changed, not the test | Add fixture paths (`tests/fixtures/`, `*.golden`) to the path list |
| Tests outside the path patterns | A test file named `checks.py` | Match on your repository's real test layout |
| A reason that is not true | "AC3 changed" when it did not | The launcher checks the cited criterion |

## From my harness

The launcher in my setup runs a small tool of this shape over every delegated branch before merge:

1. It lists modified and deleted pre-existing tests on the branch.
2. It matches each one against the branch's `Test-edit-reason:` trailers.
   - The tool's own documentation warns about the trailer-block rule above: a reason written in the body, or pushed out of the final block by a commit tool that appends its own trailers, is invisible to the trailer query.
3. It prints the unmatched ones with their removed lines.
4. The launcher treats each unmatched edit as a review finding.
   - It does not block the merge on its own, because a block would also stop legitimate edits.

The tool is layer 3 of the harness (a script that flags at the moment of review); the testing rule's "do not weaken a test" line is layer 1 above it.

## Sources

- [Escalation channels study (test-file modification flag)](https://arxiv.org/abs/2608.29460), verified 2026-11-09.
- [EvilGenie (test file edit detection)](https://arxiv.org/abs/2511.21654), verified 2026-10-08.
- [git interpret-trailers (trailer block position)](https://git-scm.com/docs/git-interpret-trailers), verified 2026-11-09.
- [git log pretty formats (`%(trailers:...)`)](https://git-scm.com/docs/pretty-formats), not re-verified for this page.

{% include page_nav.html %}
