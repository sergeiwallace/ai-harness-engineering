---
title: "Orchestration and workflows"
nav_order: 20
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Orchestration and workflows

As of 2026-11-09.

Stop 7 on the talk path, about 2 minutes.

- Two kinds of orchestration. Prompt-driven: a coordinator session or lead agent directs others through briefs and agent profiles. Programmatic: a script or graph holds the plan, as in Claude Code's dynamic workflows or LangGraph.
- `/goal` and `/loop` keep work going. Every loop needs an exit condition and a limit.
- `/workflows` watches, pauses and saves dynamic workflows, the scripts Claude writes to run many sub-agents. A workflow starts when you ask for one or use the `ultracode` keyword.

## Pages for this topic

- [Orchestration in detail]({{ '/sections/10-orchestration/detail.html' | relative_url }})
- [Advanced orchestration topics]({{ '/sections/10-orchestration/advanced.html' | relative_url }})

{% include page_nav.html %}
