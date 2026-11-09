---
title: "How much of the diff to read"
parent: "Code review"
nav_order: 2
---

{% include page_nav.html %}

# How much of the diff to read
{: .no_toc }

As of 2026-11-09. Part of [Code review]({{ '/sections/13-code-review/' | relative_url }}).

When an agent hands back a large diff, there is a strong pull toward one of two shortcuts: read the biggest file and trust the rest, or read every line with equal care until attention runs out. The first leaves the unread files unchecked, and the second runs out of attention. The research behind this page recommends complete accounting with variable depth: every changed file is listed and classified, every hand-written behaviour change is read, and extra reasoning goes to the hunks where the risk is. This page gives the definitions, the six-step review sequence, the risk triggers, and the one narrow case where sampling is safe.

<figure>
  <img src="{{ '/assets/images/13-code-review/review-depth.svg' | relative_url }}" width="100%" alt="Sequence diagram with five lanes: reviewer, inventory, diff, surrounding code and gates. The reviewer lists 14 changed files and compares them to the task. It reads all code and test hunks, finds an auth hunk and a loosened test, traces the auth callers and finds a skipped check. It runs pinned tests and scans, then reconciles the inventory against the findings.">
  <figcaption>The six-step sequence applied to a 14-file agent branch. Made with archify.</figcaption>
</figure>

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## Four terms

| Term | Meaning |
|:--|:--|
| Complete accounting | Every changed file is listed and classified: code, test, configuration, generated, vendored, lockfile, data, documentation or deletion. Nothing is silently skipped. |
| Full semantic review | Every hand-written hunk that changes behaviour, every deletion, and every test or configuration change is read and understood. |
| Targeted deep review | Extra reasoning and context retrieval on high-risk hunks, after the full pass. |
| Spot-checking | Reading a sample and inferring the rest is fine. Safe only when another control covers the rest exhaustively. |

With these terms, "read everything" means complete coverage of behaviour changes, and "targeted" means variable depth on top of complete coverage.

## What the size evidence says

### The classic numbers

| Source | Finding | How far it transfers |
|:--|:--|:--|
| A Cisco case study | Reviews under 200 lines found more defects per line | Observational; assumed defect density was constant |
| Kemerer and Paulk | Defect removal fell above about 200 lines per hour of review | About human self-review pace |
| Qt, VTK and ITK releases | Higher review coverage linked with fewer post-release defects in three of four releases | Coverage helps; alone it is not enough |
| Google's guidance | About 100 lines is reasonable; 1,000 is usually too large | Mature practice, not an experiment |

The often-repeated "200 to 400 lines in 60 to 90 minutes finds 70% to 90% of defects" could not be traced to a peer-reviewed source and is best left out of policy.

### Reviewer agents and diff size

A reviewer agent does not tire, but its accuracy still drops on large diffs:

- Attention spreads thinner over a long context.
- Tool output (file reads, test logs) piles up and competes with the diff.
- The reasoning budget for the review runs out before the last files.
- Wider diffs create more relationships to track and more chances to miss a hunk.

The human size numbers are a warning for agents, with no measured threshold. The research's operating rule:

| Hand-written change | What to do |
|:--|:--|
| Under about 400 lines | Review it whole in one context |
| Over about 400 lines | Split the change, or review coherent slices in separate fresh contexts |
| Any size with a risk trigger | Add a deep pass on the triggered hunks |

## The review sequence

### 1. Inventory

Generate the list with git:

```bash
git diff --stat --find-renames origin/main...HEAD
git diff --name-status origin/main...HEAD
```

Classify each path. Reject paths the task did not declare.

### 2. Check intent and blast radius

Compare the inventory to what the task asked for:

- Files the task names that the diff does not touch.
- Public interfaces the diff changes, and their callers outside the diff.

```bash
git grep -n "export_items(" -- ':!tests' | grep -v -f <(git diff --name-only origin/main...HEAD)
```

The command lists callers of a changed function in files the branch did not touch. Also list what should have changed but did not, which the diff cannot show: a config default, a doc page, a second code path.

### 3. Read every behaviour change

Read, in this order:

1. Every test change, so the reviewer knows what the oracle now claims.
2. Every hand-written hunk that changes behaviour.
3. Every deletion.
4. Every configuration change.

Record what was excluded and what verifies it, for example "2 generated files: regenerated with `make schema`, zero diff".

### 4. Deepen by risk

Where a risk trigger fires (next section), trace beyond the hunk:

| Trigger | What to trace |
|:--|:--|
| Permissions or authentication | Every caller of the changed check, and what runs if it is skipped |
| Migration | Forward and rollback on a copy of real data |
| Concurrency, retries, timeouts | What happens when two runs overlap or one is killed mid-way |
| Swallowed errors | What the caller sees when the error happens |

### 5. Run independent evidence

Run the checks yourself, in a clean environment, and compare the results with the author's report:

- Pinned tests, build, type and lint checks.
- Security and dependency scans.
- For a bug fix, red then green: the regression test fails on the base commit and passes on the branch.
- Regeneration of generated files, with a zero-diff check.

```bash
git switch --detach origin/main && pytest tests/test_export.py::test_csv_quotes   # expect FAIL
git switch --detach HEAD@{1}     && pytest tests/test_export.py::test_csv_quotes   # expect PASS
```

### 6. Reconcile

Before approving, check the whole review once more:

1. Re-read the inventory and confirm every file has a classification and a verdict.
2. Re-read the test diff.
3. Close or carry each open finding.
4. Check that no later fix commit invalidated an earlier approval.

## Risk triggers

Any one of these overrides a small line count and calls for a deeper or specialist read.

<details markdown="1">
<summary>The full trigger list</summary>

- Authentication, authorisation, secrets, cryptography, privacy or untrusted input.
- CI, hooks, agent instructions, repository rules or security configuration.
- Deleted tests, loosened assertions, new skips, broad mocks, snapshot churn or coverage exclusions.
- Dependencies, lockfiles, install scripts, package names or network calls.
- Schema or data migrations, destructive operations, rollback paths or irreversible state.
- Concurrency, caching, retries, timeouts, swallowed errors or cleanup.
- Public APIs, serialisation formats, contracts between repositories or shared templates.
- Large deletions, moved files, generated output with no generator, or edits outside the declared paths.

</details>

For agent work, the second item covers the agent's own instructions, hooks and CI. A change there changes the rules every later change is judged by, so it counts as a trigger even when it is small.

## When sampling is safe

In this case the sample is for diagnosis, and a deterministic verifier provides the coverage. Sampling is defensible only when all of these hold:

| # | Condition |
|:--|:--|
| 1 | The transformation was declared before it ran and has few degrees of freedom (a rename, a format change) |
| 2 | The allowed paths and the expected file count can be checked by a machine |
| 3 | A trusted tool can regenerate the output, or a verifier checks every instance |
| 4 | Tests, CI and the verifier itself are outside what the worker could edit |
| 5 | The sample is stratified: first, last, outliers, one per shape, every exception |
| 6 | The reviewer records what was sampled and which gate covered the rest |
| 7 | Any mismatch ends the exception: read everything or rerun the transform |

### Worked example: a 60-file rename

An agent renames `load_cfg` to `load_config` across 60 files. The checks:

| Check | Command or action | Covers |
|:--|:--|:--|
| The changed files are exactly the callers | `git grep -l load_cfg origin/main` matches the changed-file list | All 60 |
| No old name remains | `git grep -n load_cfg` returns nothing | All 60 |
| Build and tests pass | The merge gate | All 60 |
| Read for diagnosis | The definition, the first and last call sites, one test, and the one file where the name appeared inside a string | 5 files |

The grep and the build cover all 60 files; the five read files are for diagnosis.

## From my harness

The rule I review delegated work by is the working rule in the [code review overview]({{ '/sections/13-code-review/#shared-blind-spots' | relative_url }}): trust scales with independent verifiability and constrained freedom. Two cases from my own work:

- An agent's mechanical sweep across many repositories was accepted.
  - Every output was checked by an independent deterministic verifier.
- An agent's claim that a failing test was "pre-existing" was not accepted at first.
  - The test was run on the base commit before anyone believed it.

## Sources

- Google, what to look for in a code review: <https://google.github.io/eng-practices/review/reviewer/looking-for.html>, verified 2026-07-26.
- Google, small changes: <https://google.github.io/eng-practices/review/developer/small-cls.html>, verified 2026-07-26.
- Modern code review and software quality (Qt, VTK, ITK): <https://rebels.cs.uwaterloo.ca/papers/emse2016_mcintosh.pdf>, verified 2026-07-26.
- GitHub rulesets (path-specific required reviewers): <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets>, verified 2026-07-26.

{% include page_nav.html %}
