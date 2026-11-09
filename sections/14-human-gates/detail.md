---
title: "Detail"
parent: "Human gates, decisions and alignment"
nav_order: 1
---

# Human gates in detail

As of 2026-11-09.

## Purpose

Show where the person stays in the loop, and how to keep agents aligned with what the person asked for.

## Key points

- Decide in advance which actions need a person: publishing, sending, deleting, spending money and architectural choices. Let agents decide contained technical calls, and record why.
- Asking the person too often is also a failure. Each question costs their time and stops the work.
- Agents drift from the original request as work passes through several hands. Keep the original words, and check each result against them.
- A blocked agent investigates first, and asks only if a gate still applies.
- `/permissions` with auto mode: Sets allow, ask and deny rules; auto mode lets a classifier approve routine actions in long runs.

## Examples from my own harness

- A decision procedure with an authority test that decides who decides.
- The person's original request stored with a hash, carried into every brief, and a check that fails a decision that drops something the person asked for.
- Held decision documents that collect every question for me in one review queue.

To start: a written list of the actions an agent must never take alone.

## Pitfalls and lessons

- Gating too much leaves agents waiting on calls they could make. Push contained decisions to the agents and keep a record of each one.
- Reflexivity example: the drift check exists because an agent's decision once dropped a feature the request had named, with every other gate green.

## Sources

From the author's own harness, described conceptually.
