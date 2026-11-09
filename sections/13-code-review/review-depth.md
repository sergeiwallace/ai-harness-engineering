---
title: "How much of the diff to read"
parent: "Code review"
nav_order: 2
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# How much of the diff to read
{: .no_toc }

When an agent hands back a large diff, there are two common shortcuts, and both leave gaps:

| Shortcut | Gap |
|:--|:--|
| Read the biggest file and trust the rest | The unread files are unchecked |
| Read every line with equal care | Attention runs out, often before the risky hunks |

The research behind this page recommends complete accounting with variable depth: every changed file is listed and classified, every hand-written behaviour change is read, and extra reasoning goes to the hunks where the risk is.

This page covers:

1. four terms that make the distinction precise;
2. what the evidence on review size says, for people and for agents;
3. the six-step review sequence, with the commands for each step;
4. the risk triggers that call for a deeper read;
5. the one narrow case where sampling is safe;
6. pitfalls.

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
| A Cisco case study (2006) | Reviews under 200 lines found more defects per line | Observational; assumed defect density was constant across sizes |
| Kemerer and Paulk (2009) | Defect removal fell above about 200 lines per hour of review | About deliberate human review pace |
| Qt, VTK and ITK releases (2016) | Higher review coverage linked with fewer post-release defects in three of four releases | Coverage helps; alone it is not enough |
| Google's public guidance | About 100 lines is reasonable; 1,000 is usually too large | Mature practice, not an experiment |

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

Generate the list with git, never by hand:

```bash
git diff --stat --find-renames origin/main...HEAD
git diff --name-status --find-renames origin/main...HEAD
```

Classify each path. For the 14-file branch in the diagram:

| Class | Files | How it is reviewed |
|:--|--:|:--|
| Code | 9 | Read every behaviour-changing hunk |
| Test | 3 | Read first; run the [test-edit check]({{ '/sections/12-testing/test-edit-detection.html' | relative_url }}) |
| Generated | 2 | Regenerate and confirm a zero diff, instead of reading |

Then compare the list with the paths the task declared. A changed path outside them is a finding before any reading starts.

### 2. Check intent and blast radius

Compare the inventory to what the task asked for:

- Files the task names that the diff does not touch.
- Public interfaces the diff changes, and their callers outside the diff.

```bash
# Callers of a changed function, in files the branch did not touch (a rough filter)
git grep -n "export_items(" -- ':!tests' | grep -v -F -f <(git diff --name-only origin/main...HEAD)
```

Also list what should have changed but did not, which the diff cannot show: a config default, a doc page, a second code path.

### 3. Read every behaviour change

Read, in this order:

1. Every test change, so the reviewer knows what the oracle now claims.
2. Every hand-written hunk that changes behaviour.
3. Every deletion.
4. Every configuration change.

Record what was excluded and what verifies it, for example "2 generated files: regenerated with `make schema`, zero diff".

### 4. Deepen by risk

Where a [risk trigger](#risk-triggers) fires, trace beyond the hunk:

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

The red-then-green check, with the base in its own worktree so the branch checkout is untouched:

```bash
git worktree add --detach /tmp/review-base origin/main
cp tests/test_export.py /tmp/review-base/tests/
( cd /tmp/review-base && pytest -q tests/test_export.py::test_csv_quotes )   # expect FAIL
pytest -q tests/test_export.py::test_csv_quotes                             # expect PASS
git worktree remove --force /tmp/review-base
```

[The bug-fix regression test]({{ '/sections/12-testing/regression-tests.html#where-each-rule-is-enforced' | relative_url }}) turns the same check into a gate.

### 6. Reconcile

Before approving, check the whole review once more:

1. Re-read the inventory and confirm every file has a classification and a verdict.
2. Re-read the test diff.
3. Close or carry each open finding.
4. Check that no later fix commit invalidated an earlier approval.

## Risk triggers

Any one of these overrides a small line count and calls for a deeper or specialist read:

| Area | Examples | Why it needs depth |
|:--|:--|:--|
| Security | Authentication, authorisation, secrets, cryptography, privacy, untrusted input | A small change can open a large hole; see [security review]({{ '/sections/13-code-review/#security-review' | relative_url }}) |
| Rules and automation | CI, hooks, agent instructions, repository rules, security configuration | These change the rules every later change is judged by |
| The oracle | Deleted tests, loosened assertions, new skips, broad mocks, snapshot churn, coverage exclusions | The change may weaken its own check |
| Supply chain | Dependencies, lockfiles, install scripts, package names, network calls | New code runs at install time or reaches the network |
| State | Schema or data migrations, destructive operations, rollback paths, irreversible state | Mistakes are hard or impossible to undo |
| Timing | Concurrency, caching, retries, timeouts, swallowed errors, cleanup | Failures appear only under load or interruption |
| Contracts | Public APIs, serialisation formats, contracts between repositories, shared templates | Callers outside the diff break |
| Shape | Large deletions, moved files, generated output with no generator, edits outside the declared paths | The diff hides what actually changed |

GitHub rulesets can attach required reviewers to specific paths, so a trigger like "CI configuration" can route to a named person automatically.

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

```bash
# 1. The changed files are exactly the files that used the old name on main
diff <(git grep -l load_cfg origin/main | sed 's#^origin/main:##' | sort) \
     <(git diff --name-only origin/main...HEAD | sort)

# 2. No old name remains on the branch
git grep -n load_cfg && echo "old name still present"
```

| Check | Covers |
|:--|:--|
| The two lists match (no `diff` output) | All 60 files, and no extras |
| No old name remains | All 60 files |
| Build and tests pass in the merge gate | All 60 files |
| Read the definition, the first and last call sites, one test, and the one file where the name appeared inside a string | 5 files, for diagnosis |

`git grep` on a tree prefixes each path with `origin/main:`, which is why the first command strips it. The grep and the build cover all 60 files; the five read files are for diagnosis.

## Pitfalls

| Pitfall | Why it bites | Fix |
|:--|:--|:--|
| Spot-checking a hand-written change | Unread files are unreviewed, whatever the sample showed | Complete accounting; sample only under the seven conditions |
| Reading generated files line by line | Attention goes to the wrong place | Regenerate and diff |
| Trusting the author's "all tests pass" | The report is a claim | Run the evidence yourself in a clean checkout |
| Reviewing only what is in the diff | Missing changes are invisible there | Ask what should have changed but did not |
| One context for a 2,000-line change | The last files get the least attention | Split, or review slices in fresh contexts |

## From my harness

The rule I review delegated work by is the working rule in the [code review overview]({{ '/sections/13-code-review/#shared-blind-spots' | relative_url }}): trust scales with independent verifiability and constrained freedom. Two cases from my own work:

- An agent's mechanical sweep across many repositories was accepted.
  - Every output was checked by an independent deterministic verifier.
- An agent's claim that a failing test was "pre-existing" was not accepted at first.
  - The test was run on the base commit before anyone believed it.

## Sources

- [Cisco code review case study](https://static0.smartbear.co/support/media/resources/cc/book/code-review-cisco-case-study.pdf), verified 2026-07-26.
- [Kemerer and Paulk, design and code reviews (PSP data)](https://sites.pitt.edu/~ckemerer/PSP_Data.pdf), verified 2026-07-26.
- [Modern code review and software quality (Qt, VTK, ITK)](https://rebels.cs.uwaterloo.ca/papers/emse2016_mcintosh.pdf), verified 2026-07-26.
- [Google, what to look for in a code review](https://google.github.io/eng-practices/review/reviewer/looking-for.html), verified 2026-07-26.
- [Google, small changes](https://google.github.io/eng-practices/review/developer/small-cls.html), verified 2026-07-26.
- [GitHub rulesets (path-specific required reviewers)](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets), verified 2026-07-26.

{% include page_nav.html %}
