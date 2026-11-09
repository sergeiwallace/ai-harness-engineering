---
title: "Verifying review findings"
parent: "Review rounds and stop rules"
grand_parent: "Code review"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Verifying review findings
{: .no_toc }

A review finding is a claim about the code, and some claims are wrong:

- the cited code path is never reached;
- the "missing" check happens upstream;
- the race cannot occur in the way the code is called.

Fixing a false finding costs time and adds code for a case that cannot happen. Verifying a finding means handing it to separate agents that try to confirm or refute it against the code, with evidence, before anyone fixes it.

This page covers how verification works, what counts as evidence, a verifier brief you can reuse, a worked example, when it is worth the cost, and the edge cases and pitfalls.

<figure>
  <img src="{{ '/assets/images/13-code-review/finding-verify.svg' | relative_url }}" width="100%" alt="Sequence diagram with five lanes: reviewer, verifier A, verifier B, repository and owner. The reviewer sends finding 4, that a null user crashes the export, to both verifiers. Verifier A traces the callers of the export and finds that the auth middleware rejects a null user first. Verifier B writes a test with a null user, and the test passes with no crash. Both report refuted to the owner, with the caller trace and the passing test, and the finding is dropped.">
  <figcaption>Two verifiers refute one finding with two kinds of evidence. Made with archify.</figcaption>
</figure>

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## How it works

### Steps

1. The reviewer produces findings in a fixed format.
   - File and line, the claim, why it matters, and a check that would confirm it.
2. Each finding worth verifying goes to one or more verifier agents in fresh contexts.
   - They see the finding and the code, and not the reviewer's reasoning.
3. Each verifier returns confirmed, refuted or unclear, with evidence.
4. Only confirmed findings, and unclear ones above a severity bar, reach the owner.

Claude Code's team runs this shape on its own code review: separate agents vote on each finding before a person sees it.

### What counts as evidence

| Evidence | Strength | Example |
|:--|:--|:--|
| A test that fails, or passes, on the claim | Strongest | A test with a null user that crashes, or does not |
| A trace through the actual callers | Strong | "The only caller is behind auth middleware that rejects null users" |
| A quote of the exact code | Medium | The lines that do or do not perform the check |
| Reasoning without reference to the code | Not evidence | "This looks like it could fail" |

### A verifier brief

```text
You are verifying one code review finding. You did not write the code or the finding.

Finding: review/finding-4.md
Code: this checkout, read-only. Write any test you need under /tmp/verify-4/, not in the repo.

Decide CONFIRMED, REFUTED or UNCLEAR.
Attach evidence: a test you ran and its output, or a trace through the real callers
with file:line for each step. A verdict without a test or a trace counts as UNCLEAR.
```

The read-only checkout and the separate scratch directory keep the verifier from changing the code it is judging.

## Worked example: one finding, two verdicts

```text
Finding 4 (must-fix): export.py:88 dereferences user.id without a null check.
  A request with no user crashes the export.
  Confirm by: calling /export without a session.
```

```text
Verifier A: REFUTED
  /export is registered in routes.py:31 behind require_session().
  require_session() returns 401 before the handler runs when user is None.

Verifier B: REFUTED
  Added test_export_without_session: GET /export with no cookie.
  Result: 401, handler not entered. Test passes on the current branch.
```

The finding is dropped. Without verification, the likely fix was a null check in the handler: dead code for a case that cannot occur, which is the over-engineering described on the [review rounds page]({{ '/sections/13-code-review/review-loops.html#over-engineering-from-adversarial-review' | relative_url }}).

## When it is worth the cost

Verification costs tokens for every finding it checks. The table shows where a false positive costs the most:

| Finding | Verify? | Why |
|:--|:--|:--|
| Must-fix correctness or security claim | Yes; two verifiers is an option | A wrong fix here adds risk |
| Claim about code outside the diff | Yes | The reviewer is most likely to be wrong where it saw least |
| Performance claim | Yes, with a measurement | Intuition about speed is often wrong |
| Style or naming | No | Cheaper to accept or ignore |
| A finding the reviewer already proved with a failing test | No | The evidence is attached |

### Number of verifiers

| Setup | What it catches | Cost |
|:--|:--|:--|
| One verifier, required to attach a test or trace | Most false findings | One extra pass per finding |
| Two verifiers with different methods (one traces, one tests) | More false findings, and disagreements worth a look | Two passes per finding |

When two verifiers disagree, the finding goes to the owner marked unclear, with both pieces of evidence attached.

### Edge cases

| Situation | What to do |
|:--|:--|
| The finding cannot be reproduced in the test environment (it needs production data or a live service) | Mark it unclear; the owner decides whether to accept the risk or build a reproduction |
| The confirming test is flaky | Run it several times; a test that fails only sometimes is evidence of a timing issue, not of the claimed defect |
| The finding is about a missing behaviour, not a bug | Check it against the acceptance criteria; if no criterion asks for it, it is optional |
| Both verifiers confirm, but the fix would break a criterion | The owner rules; the criteria come first |

## The author as verifier

Letting the author verify findings against its own code is the cheapest setup and the weakest:

- The author holds the reasoning that produced the code, so it tends to argue for it.
- Research on model self-correction (ICLR 2024) found that self-correction of reasoning without external feedback can fail or make results worse.
- Separate ICLR 2024 work found that critique using tools, such as running code or searching, improved results.

So the verifier is best a separate agent that runs a tool (a test, a trace) against the code. [Cross-family review]({{ '/sections/13-code-review/cross-family-review.html' | relative_url }}) rests on the same reasoning.

## Pitfalls

| Pitfall | Why it bites | Fix |
|:--|:--|:--|
| A verifier that only reasons | Its verdict is another opinion | Require a test or a trace |
| A verifier with write access to the repo | It may "fix" the code while judging it | Read-only checkout, scratch directory for tests |
| Verifying every style note | Tokens spent where a false positive costs nothing | Verify must-fix and outside-the-diff claims |
| Dropping a refuted finding without a record | The next round raises it again | Record it, with the evidence, in the findings ledger |

## From my harness

My harness does not yet run a separate verifier per finding by default. What it does today:

1. Findings are sorted by severity.
2. The session that owns the change checks each must-fix finding against the code before acting on it.
   - A reviewer from the other family is sometimes wrong about what a call path reaches.
3. A refuted finding is recorded with its evidence.

A verifier step per must-fix finding would be the next layer, built into the review workflow so it runs without being asked, following the design on this page.

## Sources

- [Code review at Anthropic (separate agents verify each finding)](https://claude.dev/blog/solving-code-review-at-anthropic/), verified 2026-11-09.
- [Claude Code best practices (fresh-context reviewer)](https://code.claude.com/docs/en/best-practices), verified 2026-10-08.
- [Large language models cannot self-correct reasoning yet (ICLR 2024)](https://proceedings.iclr.cc/paper_files/paper/2024/hash/8b4add8b0aa8749d80a34ca5d941c355-Abstract-Conference.html), verified 2026-07-26.
- [CRITIC: self-correction with tool-interactive critiquing (ICLR 2024)](https://proceedings.iclr.cc/paper_files/paper/2024/hash/fef126561bbf9d4467dbb8d27334b8fe-Abstract-Conference.html), verified 2026-07-26.

{% include page_nav.html %}
