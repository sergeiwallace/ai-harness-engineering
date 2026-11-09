---
title: "Detail"
parent: "Code review"
nav_order: 1
---

{% include page_nav.html %}

# Code review in detail

As of 2026-11-09.

## Purpose

Cover how to review agent output with agents and people, and the current practices and pitfalls.

## Key points

- Diff review means reading what changed. At high volume nobody reads every change, so automated checks and agent review decide most of what lands.
- Claude Code reviews a diff or pull request on request (`/review`) and can review every pull request automatically.
- Cross-family review: a model from another family reviews the change, because a model shares blind spots with its own family.
- Have a separate agent verify each review finding. Claude Code's team has separate agents vote on each review finding before a person sees it. After it capped review rounds and limited re-review to the new diff, more internal pull requests passed after one round (65% to 78%).
- An agent's approval does not replace a person's where a person must approve. Claude Code's team puts it this way: "An agent's approval never counts as a person's."
- Merging at scale: when many agents finish at once, merges queue up. Route them through one owner or a merge queue, run the affected tests on the merged result, and measure where the time goes.
- Open-ended review is a pitfall. Claude Code's best-practices page warns: "A reviewer that's asked to find problems will always find *something*. If you fix every finding, you end up with extra abstraction layers and tests for cases that can't happen."
- `/code-review` (alias `/review`), `/security-review`: Reviews a branch or pull request in a separate context for bugs or security risks.

## Examples from my own harness

- Codex reviews what Claude writes, and the reverse, and the review family is recorded.
- An audit skill that reviews in rounds and sorts findings into must-fix, worth considering, and take it or leave it.
- Review briefs that pair the request to look for improvements with a statement that "no change warranted" is a valid result.
- A sweep that reviews unmerged branches and ships the ready ones.

To start: have a model from another family review one change, and check which of its findings were real.

## Pitfalls and lessons

- A review loop with no fixed acceptance criteria answers whichever reviewer spoke last and does not converge. Fix the criteria first, and stop when findings stop being new.

## Sources

- `/review`: <https://code.claude.com/docs/en/commands.md>, verified 2026-11-08.
- Automatic review of every pull request: <https://code.claude.com/docs/en/platforms.md>, verified 2026-11-08.
- "will always find something": <https://code.claude.com/docs/en/best-practices.md>, verified 2026-11-08.
- Claude Code review verification and the 65% to 78% figure: <https://claude.dev/blog/solving-code-review-at-anthropic/>, verified 2026-11-09.

{% include page_nav.html %}
