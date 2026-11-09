---
title: "Advanced"
parent: "Agent profiles, sub-agents and delegation"
nav_order: 2
---

# Advanced delegation topics

As of 2026-11-09.

- Each commit records the delegation in trailers: family and role, tier, model, effort, persona, worktree and session.
- A worktree is removed only after an audit proves its content reached the main branch, and a live process inside it blocks removal.
- The Claude Code nesting limit is set by `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`.
