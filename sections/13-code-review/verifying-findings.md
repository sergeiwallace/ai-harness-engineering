---
title: "Verifying review findings"
parent: "Review rounds and stop rules"
grand_parent: "Code review"
nav_order: 1
---

{% include page_nav.html %}

# Verifying review findings
{: .no_toc }

As of 2026-11-09. Part of [Review rounds and stop rules]({{ '/sections/13-code-review/review-loops.html' | relative_url }}), in [Code review]({{ '/sections/13-code-review/' | relative_url }}).

A review finding is a claim about the code. Some claims are wrong: the cited path is never reached, the "missing" check happens upstream, the race cannot occur. Fixing a false finding costs time and adds code for a case that cannot happen. Verifying a finding means handing it to separate agents that try to confirm or refute it against the code, with evidence, before anyone fixes it. This page covers how verification works, what counts as evidence, and when it is worth the cost.

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

1. The reviewer produces findings in a fixed format: file and line, claim, why it matters, and a check that would confirm it.
2. Each finding worth verifying goes to one or more verifier agents in fresh contexts. They see the finding and the code, not the reviewer's reasoning.
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

> Require a trace or a test with each verdict, and treat a verdict with neither as unclear.

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

Verification costs tokens for every finding it checks. The table shows where a false positive costs the most.

| Finding | Verify? | Why |
|:--|:--|:--|
| Must-fix correctness or security claim | Yes, two verifiers | A wrong fix here adds risk |
| Claim about code outside the diff | Yes | The reviewer is most likely to be wrong where it saw least |
| Performance claim | Yes, with a measurement | Intuition about speed is often wrong |
| Style or naming | No | Cheaper to accept or ignore |
| A finding the reviewer already proved with a failing test | No | The evidence is attached |

### Number of verifiers

One verifier with a required test or trace catches most false findings. Two that use different methods (one traces, one tests) catch more and disagree usefully. When they disagree, the finding goes to the owner as unclear with both pieces of evidence.

## The author as verifier

The author has every reason to refute a finding and holds the reasoning that produced the code. [Cross-family review]({{ '/sections/13-code-review/cross-family-review.html' | relative_url }}) rests on the same reasoning. Earlier research on model self-correction points the same way: self-correction without external feedback can fail or make results worse, while critique that uses tools and external feedback helps.

## From my harness

My harness does not yet run a separate verifier per finding by default. What it does today is the cheaper half: findings are sorted by severity, and the session that owns the change checks each must-fix finding against the code before acting on it, because a reviewer from the other family is sometimes wrong about what a call path reaches. A verifier step per must-fix finding would be the next layer, following the design on this page.

## Sources

- Code review at Anthropic (separate agents verify each finding): <https://claude.dev/blog/solving-code-review-at-anthropic/>, verified 2026-11-09.
- Claude Code best practices (fresh-context reviewer): <https://code.claude.com/docs/en/best-practices>, verified 2026-10-08.

{% include page_nav.html %}
