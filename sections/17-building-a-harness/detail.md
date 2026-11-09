---
title: "Detail"
parent: "Building your own harness"
nav_order: 1
---

{% include page_nav.html %}

# Building a harness in detail

As of 2026-11-09.

## Purpose

Bring the pieces together: how to start, how to grow a harness with systems thinking and reflexivity, and how to keep it from growing out of control.

## Key points

- Start from friction. Note each time an agent fails, stalls or needs the same correction twice, and work through that list.
- Think of the model, its context, its tools, its checks and yourself as one system. For each item, find the part that let it through, and fix that part: a missing piece of context, a missing check, an unclear template or a missing tool.
- Prefer a fix that a program enforces. Use the ladder in [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}): start with the softest step that works, and move a rule up when agents keep breaking it. Instructions in context files drift as sessions grow long and busy. Hooks, scripts, tests and gates do not.
- Budget for the cost of each guard. Check how often it blocks correct work, give it a time limit, and keep its refusal message useful.
- Make prompting easier for yourself and for the agents. A skill or template that turns a long instruction into a short one helps both, because agents write most of the prompts in a large harness.
- Let agents do the improvement work. An agent can file the bug, write the fix and add the check.
- Keep the person in the loop for outward, destructive and architectural decisions (see [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }})), and hand the rest to the agents.
- Self-improving harness research points the same way: mine failures, propose small bounded edits, keep only changes that do not regress, and keep the evaluator outside the thing being changed (not yet verified against the primary source). One study found automatic harness changes did no better than simply running the model more.

## Complexity creep

- Every fix adds a mechanism. Over time a harness can become a patchwork of overlapping rules, hooks and scripts that nobody can hold in their head, where a change in one place breaks another.
- Prune and consolidate. When two rules or hooks cover the same failure, merge them into one.
- Retire mechanisms when models improve. A rule or check that compensated for an older model can waste tokens or make a newer one worse (see [Trends and model changes]({{ '/sections/18-trends/' | relative_url }})).
- Measure whether each mechanism still pays for its cost: how often it fires, how often it blocks correct work, and how much time and context it uses.
- Prefer a few strong mechanisms over many weak ones. A single gate that checks the result can replace several instructions that try to prevent the problem.
- Keep reusable parts separate from personal settings, so each part can be understood and tested on its own.

## Examples from my own harness

- A first week, as one possible start: one `CLAUDE.md`, one skill for the prompt you type most, one hook for the rule an agent broke, and one rule for what needs your approval.
- The instruction-to-guard case studies in [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) show the friction loop end to end.

## Pitfalls and lessons

- Each added rule and tool costs tokens and time on every run. Remove the ones that no longer help.
- What works depends on the model. A rule that helped one model generation can hurt the next.

## Sources

- Self-improving harness research (mine failures, propose small bounded edits, keep only changes that do not regress, keep the evaluator outside the thing being changed): not yet verified against the primary source.
- Automatic harness changes doing no better than running the model more: <https://arxiv.org/abs/2607.12227>, verified 2026-11-09.

{% include page_nav.html %}
