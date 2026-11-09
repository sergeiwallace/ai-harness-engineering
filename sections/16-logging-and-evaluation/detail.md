---
title: "Detail"
parent: "Logging, observability and evaluation"
nav_order: 1
---

{% include page_nav.html %}

# Logging and evaluation in detail

As of 2026-11-09.

## Purpose

Cover how to see what the harness is doing, and how to tell whether a change helped.

## Key points

- Log what agents do: commands run, files changed, time spent and cost.
- Measure the harness itself. In a busy setup, most of the time to merge can be waiting for the final test run. Measure before optimising.
- Put a budget check before any job that makes paid model calls, so unplanned spend stops before the bill shows it.
- Anthropic: "When we evaluate 'an agent,' we're evaluating the harness and the model working together." It suggests starting with 20 to 50 simple tasks drawn from real failures, and keeping capability evals separate from regression evals.
- OpenAI recommends eval-driven development and running evals on every change.
- Evaluate one change at a time: fix the model and the task set, change one thing, run several times and compare.
- Claude Code's `claude plugin eval` compares a plugin's results against a run without it.
- Public benchmarks have errors. About 30% of one benchmark's tasks were found broken (not yet verified against the primary source).
- `/insights`: Writes a report on your recent sessions, friction points included, for improving your setup.

## Examples from my own harness

- Each commit records the model, effort and agent that produced it.
- Per-subsystem logs as JSONL ledgers, and spend estimates.

To start: ten tasks taken from your own past failures, run before and after one harness change.

## Pitfalls and lessons

- Without a baseline, you cannot tell whether a harness change helped.
- Slow gates are a cost too. Measure what each gate costs, and give it a budget.

## Sources

- Anthropic on agent evals: <https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents> (2026-01-09), verified 2026-11-08.
- OpenAI evaluation best practices: <https://developers.openai.com/api/docs/guides/evaluation-best-practices>, verified 2026-11-08.
- `claude plugin eval`: <https://code.claude.com/docs/en/plugin-evals.md>, verified 2026-11-08.
- Claude Code monitoring and OpenTelemetry: unverified; the monitoring pages were not fetched.
- The broken-benchmark figure: <https://openai.com/index/separating-signal-from-noise-coding-evaluations/>; the page returned HTTP 403 to a fetch on 2026-11-09, so the figure is not yet verified.

{% include page_nav.html %}
