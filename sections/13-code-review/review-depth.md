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

A reviewer agent does not tire, but its accuracy still drops on large diffs: attention spreads thinner over a long context, tool output piles up, and reasoning budgets run out. Wider diffs create more relationships to track and more chances to miss a hunk. The human size numbers are a warning for agents, with no measured threshold. The research's operating rule: above about 400 lines of hand-written change, split the change or review coherent slices in fresh contexts rather than quietly reading less.

## The review sequence

### 1. Inventory

Generate the list with git:

```bash
git diff --stat --find-renames origin/main...HEAD
git diff --name-status origin/main...HEAD
```

Classify each path. Reject paths the task did not declare.

### 2. Check intent and blast radius

Compare the changed files to the task, the public interfaces and the callers. Also list what should have changed but did not, which the diff cannot show.

### 3. Read every behaviour change

Read every hand-written hunk that changes behaviour, every deletion, every test change and every configuration change. Record which generated or mechanical files were excluded and which tool verifies them.

### 4. Deepen by risk

Trace callers, invariants, error paths, concurrency, permissions, migrations and rollback where a risk trigger fires (next section).

### 5. Run independent evidence

Pinned tests, build, type and lint checks, security scans, red-then-green proof for a bug fix, and regeneration of generated files, all in a clean environment.

### 6. Reconcile

Re-read the inventory, the test diff and the open findings. Check that no later fix invalidated an earlier approval.

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

An agent renames a function across 60 files. The reviewer checks that the changed-file list matches a `grep` for the old name on the base commit, that the old name no longer appears anywhere, that the build and tests pass, and reads five files: the definition, the first and last call sites, one test, and the one file where the call was inside a string. All 60 are covered by the grep and the build; the five are for diagnosis.

## From my harness

The rule I review delegated work by is the working rule in the [code review overview]({{ '/sections/13-code-review/#review-of-agent-written-code' | relative_url }}): trust scales with independent verifiability and constrained freedom. An agent's mechanical sweep across many repositories was accepted after an independent check of every output; an agent's claim that a failure was "pre-existing" was not accepted until the test ran on the base commit.

## Sources

- Google, what to look for in a code review: <https://google.github.io/eng-practices/review/reviewer/looking-for.html>, verified 2026-07-26.
- Google, small changes: <https://google.github.io/eng-practices/review/developer/small-cls.html>, verified 2026-07-26.
- Modern code review and software quality (Qt, VTK, ITK): <https://rebels.cs.uwaterloo.ca/papers/emse2016_mcintosh.pdf>, verified 2026-07-26.
- GitHub rulesets (path-specific required reviewers): <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets>, verified 2026-07-26.

{% include page_nav.html %}
