---
title: "Code review"
nav_order: 23
has_children: true
has_toc: false
permalink: /sections/13-code-review/
---

{% include page_nav.html %}

# Code review
{: .no_toc }

As of 2026-11-09. Stop 9 on the talk path, about 2 minutes.

When agents write most of the code, nobody reads every change by hand. This section follows one way of handling that, which is to review in layers:

1. Deterministic gates run first.
   - Tests, lint, type checks and scans decide anything with a pass or fail result.
2. An agent reads the diff.
   - In my harness it comes from a different model family than the author.
3. Separate agents verify each finding.
   - Refuted findings do not reach the owner.
4. A person approves where policy requires one.

This overview covers each layer with a worked example and the commands that run it. It also covers two failure modes of agent review: reviewers that keep finding things, and review loops that do not converge.

<figure>
  <img src="{{ '/assets/images/13-code-review/review-pipeline.svg' | relative_url }}" width="100%" alt="Sequence diagram with five lanes: author agent, merge gate, reviewer agent from another family, verifier agents, and owner. The author pushes a branch with its criteria; the gate fails lint on one file; the author fixes and pushes again. The gate hands the diff and criteria to the reviewer, who reads every changed hunk and sends six findings to the verifiers in a fresh context. The verifiers confirm four and refute two. The owner asks for three fixes and one justification. The author asks for a re-review of the new diff only; the reviewer finds nothing new; the owner approves and merges, noting that an agent approval never counts as a required human one.">
  <figcaption>One change reviewed in layers, top to bottom. Each layer hands fewer items to the next. Made with archify.</figcaption>
</figure>

| Layer | What it does | Topic page |
|:--|:--|:--|
| Deterministic gates | Tests, lint, type checks, scans: anything with a pass or fail | This page, [deterministic gates](#deterministic-gates) |
| Complete, risk-weighted reading | Every changed file accounted for, depth where the risk is | [How much of the diff to read]({{ '/sections/13-code-review/review-depth.html' | relative_url }}) |
| Cross-family review | A reviewer from another model family reads the diff | [Cross-family review]({{ '/sections/13-code-review/cross-family-review.html' | relative_url }}) |
| Finding verification | Separate agents confirm or refute each finding | [Verifying review findings]({{ '/sections/13-code-review/verifying-findings.html' | relative_url }}) |
| Bounded rounds | Fixed criteria, re-review of the new diff, a stop rule | [Review rounds and stop rules]({{ '/sections/13-code-review/review-loops.html' | relative_url }}) |
| Human approval | A person where policy requires one | This page, [human approval](#human-approval) |

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## Where review sits in the harness

This site treats harness engineering as a systems approach: when an agent gets something wrong, the fix goes into the system, enforced by code wherever possible, and each new gap gets the same treatment. Review spans the enforcing layers of that system:

1. Instructions in context files, skills and prompts.
   - Example: a review brief that says "no change warranted is a valid result".
   - These guide the reviewer and are applied inconsistently, so they are the second line of defence.
2. Workflows and tools that build the steps in.
   - Example: a review command that always runs in a fresh context and records the reviewing family.
3. Hooks, scripts and linters that refuse or flag at the moment of action.
   - Example: a dependency check that refuses an unknown package, or a test-edit flag on the branch.
4. Validation and tests before a change lands.
   - Example: the merge gate, then agent review with verified findings, then human approval where required.

When a review finds the same kind of defect twice, the response is a new gate or check in layers 2 to 4. A new line in the reviewer's instructions is at most a second layer on top of it.

## Review of agent-written code

### Volume

Agent output outruns careful human reading:

| Measure | Figure | Source |
|:--|:--|:--|
| Careful human review pace | About 200 lines per hour before defect removal falls | Kemerer and Paulk's inspection study |
| A reasonable change size | About 100 lines; 1,000 is usually too large | Google's public review guidance |
| Agent output | A few hundred lines every few minutes per agent | Illustrative, not a measurement |

At that volume, review that depends on one person reading everything either becomes the bottleneck or turns into skimming without anyone deciding it should.

### Plausibility

Agent code tends to look right, which makes surface reading weak evidence:

- In the 2025 Stack Overflow developer survey, 66% of respondents who answered the question named solutions that are "almost right" as a frustration.
  - 45% said debugging AI-generated code takes more time.
- A USENIX Security 2025 study generated 576,000 code samples across 16 models.
  - 19.7% of the packages the models recommended did not exist.
  - A reader can notice an unfamiliar name; only resolving it against the registry proves it exists.

### Shared blind spots

A model reviewing its own output, or output from its own family, tends to rate it favourably:

- Peer-reviewed work (NeurIPS 2024) found that models recognise their own outputs at better than chance and score them higher than equal-quality outputs from others.
- The stronger a model's self-recognition, the stronger its self-preference.
- A 2026 study across 20 models found that capability did not predict low self-preference.

Using a reviewer from a [different family]({{ '/sections/13-code-review/cross-family-review.html' | relative_url }}) is one response. The research behind this section adds a working rule for any reviewer: trust a result in proportion to how independently it can be checked and how constrained the work was.

## Deterministic gates

A check that an exit code can decide runs as a gate before any reviewer reads the diff:

1. The author pushes the branch.
2. The gate runs every deterministic check (table below).
3. A red result sends the branch back to the author with the failing output.
   - No reviewer time is spent on it.
4. A green result hands the diff and the acceptance criteria to the reviewer.

The [review pipeline diagram](#code-review) at the top of this page shows this as the first lane.

### What a gate catches, and what it does not

| Gate | Catches | Does not establish |
|:--|:--|:--|
| Build, type check, lint | Syntax, types, known static rules | That the requirement is met |
| Pinned tests, run on the merged result | Regressions in known behaviour | Unknown cases; weak assertions |
| [Test-edit check]({{ '/sections/12-testing/test-edit-detection.html' | relative_url }}) | Deleted or loosened assertions, new skips | Special-casing in the code |
| Changed-path allowlist | Edits outside the declared scope | Whether in-scope edits are right |
| Dependency resolution, secret and licence scans | Invented packages, leaked keys | Safe use of a real package |
| Generator replay with a zero-diff check | Drift in generated files | Correctness of the generator |

### Worked example: a red gate

An agent's branch changes 14 files. The gate output:

```console
$ ./gate
lint ............ FAIL  src/export.py:3  F401 'json' imported but unused
deps ............ FAIL  requirements.txt  'csvkit-fast' not found on the package index
tests (14 sel.) . skipped (install failed)
gate: RED, 2 failures, returned to author
```

- The unused import is a lint finding the author fixes in seconds.
- The missing package is the hallucinated-dependency case from the [plausibility](#plausibility) evidence; the author replaces it with the standard library's `csv` module.
- The branch reaches a reviewer only after the gate is green.

## Review depth

Spot-checking the biggest file says nothing about the files no one opened. Reading every line with equal care wastes attention on generated output and lockfiles.

<figure>
  <img src="{{ '/assets/images/13-code-review/review-depth.svg' | relative_url }}" width="100%" alt="Sequence diagram with five lanes: reviewer, inventory, diff, surrounding code and gates. The reviewer lists every changed path: 14 files, 9 code, 3 test, 2 generated, with generated files regenerated and diffed instead of read. The reviewer compares the list to the task, asking what should have changed but did not. It reads all code and test hunks and finds one hunk touching auth and one loosened test. It traces the auth callers and finds a new path that skips a check. It runs pinned tests and scans, which are green, and re-checks the inventory against the findings.">
  <figcaption>Complete accounting, then depth where the risk is. Made with archify.</figcaption>
</figure>

The research behind this section recommends complete accounting with risk-weighted depth: list every changed file, classify it, read every hand-written behaviour change, and spend extra reasoning on the hunks that touch authentication, tests, migrations, deletions or shared interfaces. [How much of the diff to read]({{ '/sections/13-code-review/review-depth.html' | relative_url }}) gives the full sequence, the risk triggers, and the narrow case where sampling is safe.

## Cross-family review

Cross-family review uses a reviewer from a different model family than the author:

| Author | Reviewer | Recorded as |
|:--|:--|:--|
| Claude model | Codex model | Cross-family |
| Codex model | Claude model | Cross-family |
| Claude model | Another Claude model, fresh context | Same-family, weaker |

In my harness, Codex reviews what Claude writes and the reverse, and every merged change records which family reviewed it. Same-family review is accepted when no other family is available.

### Worked example: the review brief

```text
Review this branch against the acceptance criteria below. Read every changed hunk.
For each finding give: file and line, what is wrong, why it matters, and how to verify it.
Sort findings into must-fix, worth considering, and optional.
"No change warranted" is a valid result. Do not invent findings to have something to report.

Acceptance criteria:
AC1 ...
```

The last two lines give the reviewer permission to report nothing. Claude Code's best-practices page notes that a reviewer asked only to find problems will always find something.

### The commands

| Command | Where | What it does |
|:--|:--|:--|
| `/review` | Claude Code | Reviews a branch or pull request in a separate context |
| `/security-review` | Claude Code | Reviews the pending changes for security risks |
| Automatic pull-request review | Claude Code on GitHub | Reviews every pull request without being asked |
| `/review`, or a non-interactive run with a review brief | Codex CLI | A review from the other family, usable from a script |

[Cross-family review]({{ '/sections/13-code-review/cross-family-review.html' | relative_url }}) covers why it works, how to run it from either side, and what it does not fix.

## Verify each finding

A review finding is a claim that can be wrong. Claude Code's team describes its own setup:

- Separate agents vote on each review finding before a person sees it.
- Review rounds are capped.
- Re-review covers only the new diff.

| Measure | Before | After |
|:--|--:|--:|
| Internal pull requests passing after one review round | 65% | 78% |

### Worked example: a refuted finding

A reviewer reports: "export.py:88 dereferences `user.id` without a null check; a request with no user crashes the export." Two verifiers check it independently:

1. Verifier A traces the callers.
   - `/export` is registered behind `require_session()`, which returns 401 before the handler runs when there is no user.
2. Verifier B writes a test.
   - `GET /export` with no session returns 401, and the handler is never entered.
3. Both report "refuted" with their evidence.
   - The finding does not reach the owner, and no dead null check is added.

[Verifying review findings]({{ '/sections/13-code-review/verifying-findings.html' | relative_url }}) covers how many verifiers to use, what counts as evidence, and when verification costs more than it saves.

## Bounded rounds and stop rules

### Open-ended review

Claude Code's best-practices page warns: "A reviewer that's asked to find problems will always find *something*. If you fix every finding, you end up with extra abstraction layers and tests for cases that can't happen."

How an unbounded loop goes:

1. Round 1 finds five issues; the author fixes all five.
2. Round 2, a fresh reviewer, reads the whole change again and finds four new ones, two of them defensive checks for impossible cases.
3. Round 3 finds three more, and the change is now larger and harder to read than the first version.

Each round's reviewer sets a new target, so the loop does not converge.

### Three rules for bounded rounds

1. Fix the acceptance criteria before the first round, and review against them.
2. Re-review only the new diff after each fix round.
3. Stop when a round produces no new findings. A finding that is a sibling of an earlier one means the fix addressed an instance, not the cause.

<figure>
  <img src="{{ '/assets/images/13-code-review/review-rounds.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: owner, author agent, reviewer and findings ledger. Round 1: the owner sends the diff and fixed criteria; the reviewer logs five findings, two must-fix and three optional; the owner has the author fix the two. Round 2: the author sends the new diff only; the reviewer logs one finding that is a sibling of round 1; the owner asks for the root cause to be fixed. Round 3: new diff only; zero new findings; the ledger tells the owner to stop, noting that a clean first round is checked for whether the target was read.">
  <figcaption>Three rounds that converge. Made with archify.</figcaption>
</figure>

[Review rounds and stop rules]({{ '/sections/13-code-review/review-loops.html' | relative_url }}) goes deeper, including why adversarial review over-engineers code and how to tell a converging loop from one that is chasing its tail.

## Human approval

An agent's review can replace a person's reading, but not a person's approval where one is required. Claude Code's team writes: "An agent's approval never counts as a person's." In practice:

| Change | Who approves |
|:--|:--|
| Routine change in a repository the agent owner controls | The owner's launcher, once gates and review are green |
| Change to a shared repository with required reviewers | A person on the required-reviewer list |
| Security-sensitive, destructive or outward-facing change | A person, always |

[Human gates]({{ '/sections/14-human-gates/' | relative_url }}) covers how to decide which is which.

## Merging at scale

When many agents finish at once, merges queue up, and the merged result is a combination nobody tested. Options that help:

- Route merges through one owner or a merge queue.
- Run the affected tests on the merged result as well as on each branch.
- Measure where merge time goes before optimising it.

<details markdown="1">
<summary>Aside: evidence that review alone is not enough</summary>

Anthropic's April 2026 Claude Code postmortem describes a context-management defect that passed several human and automated code reviews, unit and end-to-end tests, automated verification and internal use. A stronger reviewer found it once given the other repositories needed for full context. A separate 2025 study demonstrated agent patches that were functionally correct, passed every test, and were still vulnerable.

</details>

## From my harness

The descriptions below are conceptual.

- Codex reviews what Claude writes, and the reverse; the commit records which model and family did the work and which reviewed it.
- An audit skill that reviews in rounds and sorts findings into must-fix, worth considering, and take it or leave it.
- Review briefs that pair "look for improvements" with "no change warranted is a valid result".
- A sweep that reviews every unmerged branch across the repositories and lands the ready ones.

To start, have a model from another family review one change and count which of its findings were real.

## Sources

- `/review` and `/security-review`: <https://code.claude.com/docs/en/commands.md>, verified 2026-11-08.
- Automatic review of every pull request: <https://code.claude.com/docs/en/platforms.md>, verified 2026-11-08.
- "will always find something": <https://code.claude.com/docs/en/best-practices.md>, verified 2026-11-08.
- Finding verification and the 65% to 78% figure: <https://claude.dev/blog/solving-code-review-at-anthropic/>, verified 2026-11-09.
- Google, small changes: <https://google.github.io/eng-practices/review/developer/small-cls.html>, verified 2026-07-26.
- Package hallucinations (19.7%): <https://www.usenix.org/system/files/usenixsecurity25-spracklen.pdf>, verified 2026-07-26.
- Self-preference in LLM evaluators: <https://papers.nips.cc/paper_files/paper/2024/hash/7f1f0218e45f5414c79c0679633e47bc-Abstract-Conference.html>, verified 2026-07-26.
- Claude Code April 2026 postmortem: <https://www.anthropic.com/engineering/april-23-postmortem>, verified 2026-07-26.
- Stack Overflow 2025 developer survey, AI section: <https://survey.stackoverflow.co/2025/ai>, verified 2026-07-26.
- Functionally correct but vulnerable agent patches: <https://neurips.cc/virtual/2025/133169>, verified 2026-07-26.

{% include page_nav.html %}
