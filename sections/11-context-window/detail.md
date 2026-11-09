---
title: "Detail"
parent: "Context window and memory"
nav_order: 1
---

# Context window in detail

As of 2026-11-09.

## Purpose

Explain how to keep long sessions useful, and what should carry over between sessions.

## Key points

- As the context window fills, an agent follows instructions less reliably. Treat this as something to watch, because it changes as models improve.
- My own practice is to compact at about 50 to 60 percent of the context window. Other practitioners report a sweet spot below about 300k tokens (unverified practitioner reports).
- My observation is that Codex compacts at about 250k tokens by default, and that going past that costs more usage or API spend. Unverified: OpenAI's configuration reference says the compaction threshold uses "model defaults" and gives no number, and its pricing page says subscription use of one current model "has no long-context surcharge". Check current pricing for your plan.
- Claude Code compacts automatically near the limit. `/compact <focus>` steers the summary, and the threshold can be set. The root `CLAUDE.md`, rules without paths and auto memory are loaded again after compaction.
- Write durable state to files before compacting.
- Memory files carry lessons between sessions.
- A task tracker outside the session holds the real list of work.
- `/context`: Shows what is filling the context window, by category, with suggestions.

## Examples from my own harness

- A save-state skill that writes the current state before compaction, and a resume message format.
- A memory index with one line per lesson and one file per lesson, each recording the failure behind it.
- A command that reports the actual context percentage, and hooks that act on it.
- A task tracker as the record of all work; the task panel in the terminal only displays it.

To start: one notes file the agent updates before each compaction.

## Pitfalls and lessons

- An index that grows past what loads drops its oldest lines without a warning. Keep indexes short and archive old entries.
- A session can fall back to a smaller context window than expected. Check the limit you actually got.
- Reflexivity example: each lesson file records the failure behind it, so the agent can apply the general rule and not only the one phrase.

## Sources

- Compaction, `/compact`, what reloads, one-million-token models: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.
- Codex compaction threshold: <https://learn.chatgpt.com/docs/config-file/config-reference.md>, fetched 2026-11-08 (no default number given).
- Codex pricing: <https://learn.chatgpt.com/docs/pricing.md>, fetched 2026-11-08.
