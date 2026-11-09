---
title: "Advanced"
parent: "Code review"
nav_order: 2
---

{% include page_nav.html %}

# Advanced code review topics

As of 2026-11-09.

- Adversarial review, my observation: an open-ended "find problems" review can always find something, such as an edge case or an optimisation. Building all of it ahead of time can produce overly defensive, less elegant code. I have seen this discussed online and in my own work, especially with Codex and GPT models. Anthropic's prompting guide says newer models over-react to aggressive wording and tend to over-engineer, including defensive code. Its workflows page still recommends independent agents that review each other's findings.
- Review rounds with stop rules: a first round with no findings is checked for whether the reviewer read the target, and a round with nothing new ends the loop.

{% include page_nav.html %}
