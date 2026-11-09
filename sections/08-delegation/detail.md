---
title: "Detail"
parent: "Agent profiles, sub-agents and delegation"
nav_order: 1
---

# Delegation in detail

As of 2026-11-09.

## Purpose

Explain how to hand work to other agents and keep control of it.

## Key points

- A sub-agent runs in its own context and returns a summary, which keeps the lead agent's context small.
- An agent profile is a file (`.claude/agents/` in Claude Code, TOML in `.codex/agents/` in Codex) that sets a sub-agent's tools, model, permissions and more. In Claude Code, `isolation: worktree` runs it in its own git worktree.
- Sub-agents can start sub-agents, up to three levels deep by default in Claude Code, and the limit can be changed in settings. Set your own bounds as well: how many at once, and which ones may start others.
- Pick the model per task: the strongest model for decisions and synthesis, cheaper ones for bulk reading or mechanical edits.
- A brief states the goal, the inputs, the exact return format and the limits.
- Agent teams, where agents message each other directly, are experimental in Claude Code and off by default.
- `/tasks`: Lists the session's background work, including sub-agents that have finished.
- `/subtask`, `/fork`: `/subtask` starts a sub-agent that inherits the conversation; `/fork` copies the session into a separate background session.

## Examples from my own harness

- Agent profiles by role: analyst, architect, auditor, engineer, planner, researcher and reviewer. Read-only roles have no write tools.
- A guard that refuses to launch a writing sub-agent without its own worktree.
- A check that reads the machine's load before each launch, so many agents at once do not slow every session.
- Codex reviews what Claude writes (see [Code review]({{ '/sections/13-code-review/' | relative_url }})), and each commit records which model and family did what.

To start: one read-only researcher profile and one brief template.

## Pitfalls and lessons

- A sub-agent's summary can drop a caveat that changes the conclusion. Read the key passages yourself.
- Agents report work they did not do: a complete fix report with passing tests while no file changed. Check the files, the commits and the test output.
- Isolation can fail: sub-agents commit onto their parent's branch, or a sub-agent working across two repositories writes into the second one's main checkout. Check where the commits landed.

![A lead agent sends a brief to each of three sub-agents. Each sub-agent commits in its own worktree and returns a summary that the lead agent reads.]({{ '/assets/diagrams/delegation.svg' | relative_url }})

## Sources

- Sub-agent profiles, `isolation`, nesting depth: <https://code.claude.com/docs/en/sub-agents.md>, verified 2026-11-08.
- Agent teams: <https://code.claude.com/docs/en/agent-teams.md>, verified 2026-11-08.
- Codex sub-agents: <https://learn.chatgpt.com/docs/agent-configuration/subagents.md>, verified 2026-11-08.
