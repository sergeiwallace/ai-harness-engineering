---
title: "Claude Code and Codex basics"
nav_order: 13
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Claude Code and Codex basics

As of 2026-11-09.

Stop 4 on the talk path, about 1.5 minutes, shared with context files, skills and templates. [Claude Code and Codex in detail]({{ '/sections/03-claude-code-and-codex/detail.html' | relative_url }}) has the full feature comparison and the command reference.

## Two clients, one shape

Claude Code (from Anthropic) and Codex (from OpenAI) are coding agents that work in a repository. You start one in a terminal, inside a project folder, and give it a task in plain language. It reads the code, runs commands, edits files and reports back. Both also run inside editors, and both have desktop and web versions that share the same engine.

This site covers Claude Code first and Codex second, because the examples come from a harness built mostly on Claude Code. Almost every idea carries over, because the two clients now have the same building blocks under different names.

| Building block | Claude Code | Codex |
|---|---|---|
| Context file | `CLAUDE.md` | `AGENTS.md` |
| Skills | Yes | Yes |
| Hooks | Yes, 33 events | Yes, 14 events |
| Sub-agents | Markdown files in `.claude/agents/` | TOML files in `.codex/agents/` |
| Keep working until a condition holds | `/goal` | `/goal` |
| Repeat a prompt on an interval | `/loop` | none |
| Summarise to free context | `/compact` | `/compact` |

### Example: moving a setup between them

Codex has an `/import` command that reads a Claude Code or Cursor setup. Someone with a working `CLAUDE.md` and a few skills can bring them into Codex without rewriting them, then adjust what does not fit.

## Commands worth knowing by name

A slash command is something you type in the session, such as `/compact`. Four of them come up throughout this site.

- `/goal <condition>` keeps the session working until a separate small model judges the condition met. Use it for "keep going until the tests pass and the branch is pushed".
- `/loop` repeats a prompt on an interval, such as "check the deploy every five minutes". Recurring tasks expire after 7 days.
- `/compact <focus>` summarises the session to free context, steered by the focus text, such as `/compact keep the list of failing tests`.
- `/rc` (`/remote-control`) lets you control a session from another device. It needs a claude.ai sign-in.

### Example: an unattended fix

You type `/goal all tests in tests/api pass and the fix is committed`. The agent works. Each time it tries to stop, a small model reads the conversation and checks the condition. If the tests still fail, the agent is sent back to work. When they pass and the commit exists, the goal is met and the session stops. You come back to a finished task.

## Tools that change how an agent waits

Claude Code's Monitor tool watches a background command or a file and feeds each new output line back to the agent as it arrives. An agent waiting for a long build no longer has to keep checking; the build's output comes to it. Monitors stop when the session ends. See [deep dive: Monitor]({{ '/sections/03-claude-code-and-codex/detail.html' | relative_url }}#monitor).

## Sign-in and availability

Some features need a claude.ai sign-in and do not work with API keys or through cloud providers such as Amazon Bedrock: Remote Control, Claude in Chrome and computer use. Dynamic workflows, `/goal`, auto mode and same-machine session messaging work on cloud providers. If you use Claude Code through work, check which route your account uses before relying on a feature. The [availability table]({{ '/sections/03-claude-code-and-codex/detail.html' | relative_url }}#availability-on-cloud-providers) lists each one.

## How to start

Start with the defaults. Run the client in one project, give it real tasks, and notice where it goes wrong. Add one piece at a time when you hit a problem the defaults do not solve: a context file first, then a skill, then a hook. Turning on many features at once makes a failure hard to trace.

## Pages for this topic

- [Claude Code and Codex in detail]({{ '/sections/03-claude-code-and-codex/detail.html' | relative_url }}): surfaces, the feature comparison, configuration files, the command reference grouped by purpose, the Monitor tool, headless runs, availability on cloud providers, and an install step that keeps every machine the same.

{% include page_nav.html %}
