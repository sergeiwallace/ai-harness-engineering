---
title: "Detail"
parent: "Trends and model changes"
nav_order: 1
---

{% include page_nav.html %}

# Trends in detail

As of 2026-11-09.

## Purpose

Cover what is changing now and how to keep a harness current.

## Key points

- Harness engineering and context engineering are named practices. Anthropic defined context engineering in September 2025, and harness engineering essays followed through 2026.
- Merge volume at frontier teams has grown to the point that the system around the agents decides most of what lands. Anthropic reported: "In May 2026, Anthropic averaged around 2,500 code changes merged per day; as of September, we merged over 30,000 per day." It added: "The bottleneck, now, is all of the developer infrastructure built for human scale: source control, builds, and CI."
- Harness setups are starting to move between tools. The Agent Skills standard is shared, and Codex can import a Claude Code setup.
- Claude Code now offers several ways to run agents in parallel, some still in preview, such as agent view and Projects.
- New model generations change what a harness needs. Instructions written for an older model can waste tokens or make results worse with a newer one.
- Re-test parts of the harness when the model changes, and remove what is no longer needed. This is the same pruning step as in complexity creep (see [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }})).

## Examples from my own harness

- When the model changed, the writing tells changed. Em dashes nearly disappeared, and bold lead-in labels, three-part lists and hedges increased, so my writing rules changed with them.

## Pitfalls and lessons

- Re-test harness rules when you change models. Some rules written for an older model make a newer one worse.

## Sources

- Context engineering: <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>, verified 2026-11-08.
- Parallel agents in Claude Code: <https://code.claude.com/docs/en/agents.md>, verified 2026-11-08.
- Codex `/import`: <https://learn.chatgpt.com/docs/developer-commands.md?surface=cli>, verified 2026-11-08.
- Anthropic merged-changes figures: <https://www.anthropic.com/research/redesigning-source-control>, verified 2026-11-09.

{% include page_nav.html %}
