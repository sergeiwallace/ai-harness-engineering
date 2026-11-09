---
title: "Detail"
parent: "Skills, commands and plugins"
nav_order: 1
---

# Skills and plugins in detail

As of 2026-11-09.

## Purpose

Explain skills, plugins and mods, when to use each kind of skill, and how to combine them.

## Key points

- A skill is a folder with instructions and optional scripts. Its short description is always in context, and its body loads only when the skill is used. In Claude Code, custom commands have been merged into skills.
- Skills follow the Agent Skills open standard, which Codex also uses. Anthropic gave the standard to the Agentic AI Foundation in October 2026.
- Who can start a skill is set in its frontmatter. By default both the person and the model can. `disable-model-invocation: true` means only the person can. `user-invocable: false` means only the model can. Codex has a similar setting, `allow_implicit_invocation: false`.
- Let the model invoke skills it should use whenever the situation fits, such as a test procedure. Make a skill person-only when it has side effects the person must choose, such as publishing, sending or merging.
- Chaining skills: Claude Code lets a person start one message with several skills, up to six. A skill's instructions can also tell the agent to run another skill. That second form is a convention; the docs do not describe it.
- A plugin is "a directory of skills, agents, hooks, MCP servers, or other components that Claude Code installs and loads as one unit", usually from a marketplace. The docs say to use one "when you want several skills, subagents, hooks, or MCP servers packaged as one unit".
- One useful pattern is a plugin whose skills, MCP servers and hooks all reuse one shared code library, so the code is written once and is not copied into each skill. I call this a plugin module. That is my informal name; Claude Code has no such term. The nearest vendor term is a "hooks module": hooks written as JavaScript functions.
- A mod is a plugin with a hooks module. Mods change how Claude Code looks and behaves.
- An enabled plugin costs context on every turn, because the name and description of each model-invocable skill and agent stay loaded.

## Examples from my own harness

- Person-only skills for actions with side effects, such as pulling from git or filing feedback. A skill that another skill or a hook calls stays model-invocable, or the chain breaks.
- Chained skills: my orchestration skill hands its launch step to my delegation skill.
- A shared library for the command-line tools inside skills, so they share options, settings and test helpers, and a fix lands once for all of them.
- A core plugin that serves the same read-only tools to Claude Code and to Codex.
- Mods that draw status panes inside the terminal session.

To start: turn the prompt you type most often into a skill.

## Pitfalls and lessons

- Skills with overlapping descriptions make the model pick the wrong one.
- A skill body loads only on use. A rule that must always hold does not belong only in a skill.
- Skills pile up. Delete the ones nobody invokes.

## Sources

- Skills and invocation settings: <https://code.claude.com/docs/en/skills.md>, verified 2026-11-08.
- Several skills in one message, up to six: <https://code.claude.com/docs/en/commands.md>, verified 2026-11-08.
- Plugins, the hooks module and mods: <https://code.claude.com/docs/en/plugins/overview.md> and <https://code.claude.com/docs/en/plugins/mods/overview.md>, verified 2026-11-08.
- Agent Skills donation: <https://www.anthropic.com/news/donating-agent-skills-to-the-agentic-ai-foundation>, verified 2026-11-08.
- Codex skills: <https://learn.chatgpt.com/docs/build-skills.md>, verified 2026-11-08.
