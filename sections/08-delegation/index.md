---
title: "Agent profiles, sub-agents and delegation"
nav_order: 18
has_children: true
has_toc: false
---

# Agent profiles, sub-agents and delegation

As of 2026-11-09.

Stop 6 on the talk path, about 4 minutes.

- A sub-agent works in its own context and returns a summary.
- Agent profiles pin a role's tools and model. Pick the model per task.
- Orchestration can be plain prompting: a lead session directs sub-agents through briefs and profiles ([Orchestration and workflows]({{ '/sections/10-orchestration/' | relative_url }}) covers the programmatic kind).
- Give each writing agent its own worktree, and bound how many agents run and how deep they nest.
- Check the files and commits, because agents report work they did not do.
- `/tasks` lists the session's background work, including sub-agents that have finished.
- `/subtask` starts a sub-agent that inherits the conversation; `/fork` copies the session into a separate background session.

## Pages for this topic

- [Delegation in detail]({{ '/sections/08-delegation/detail.html' | relative_url }})
- [Advanced delegation topics]({{ '/sections/08-delegation/advanced.html' | relative_url }})
