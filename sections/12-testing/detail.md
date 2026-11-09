---
title: "Detail"
parent: "Testing agent-written code"
nav_order: 1
---

{% include page_nav.html %}

# Testing in detail

As of 2026-11-09.

## Purpose

Cover how to test code that agents write, and the ways agents get testing wrong.

## Key points

- Test what the code is for, at its real entry point. Agents tend to write many small tests of internal functions, and those can pass while the feature is broken.
- Write the failing test before the fix, and keep it as the regression test.
- Agents sometimes edit a test to make it pass. A check that flags edits to existing tests catches this cheaply.
- End-to-end and live validation tests run the real system. A green unit suite can pass while the feature is broken.
- At scale, test time becomes the bottleneck. Select the tests a change affects, and run the full suite less often.
- Newer models check their own work, so old "verify everything" instructions can waste tokens (not yet verified against the primary source).

## Examples from my own harness

- Testing rules that load only on test files, with a behaviour-first standard.
- A bug-fix procedure: reproduce, find the root cause, fix, add the regression test.
- Live end-to-end runs that send real messages between sessions and check they arrive.
- A push gate that picks the tests to run from the diff, and a machine-wide lock so parallel test runs do not overload the machine.

To start: one end-to-end test per feature.

## Pitfalls and lessons

- A larger suite takes longer to run and does not always catch more. One open-source project deleted about 400,000 lines of its own tests, and its code coverage barely moved.
- When a gate always fails on one machine, people skip it there. Keep every gate passable wherever it runs.
- In one study, a policy line plus an approved way for the agent to report a bad test cut test tampering from 23.6% to 5.3% across eight models.

## Sources

- The test-tampering study (23.6% to 5.3% across eight models): <https://arxiv.org/abs/2608.29460>, verified 2026-11-09.
- The deleted-tests example: <https://www.explainx.ai/blog/openclaw-test-audit-skill-deleted-400k-lines-tests-2026>, verified 2026-11-09, a secondary report of the maintainer's post.
- Newer models checking their own work: not yet verified against the primary source.

{% include page_nav.html %}
