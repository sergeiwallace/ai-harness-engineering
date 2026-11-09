---
title: "Detail"
parent: "Claude Code and Codex basics"
nav_order: 1
---

# Claude Code and Codex in detail

As of 2026-11-09.

## Purpose

Show what the two main clients provide before anything is customised.

## Key points

- Claude Code and Codex are agents that work in a repository from the terminal. Both also run in editors and have desktop or web versions.
- Claude Code commands worth knowing:
  - `/goal <condition>` keeps the session working until a separate small model judges the condition met.
  - `/loop` repeats a prompt on an interval; recurring tasks expire after 7 days.
  - `/compact <focus>` summarises the session to free context, steered by the focus text.
  - `/rc` (`/remote-control`) controls a session from another device.
- Some features need a claude.ai sign-in and do not work with API keys or through cloud providers: Remote Control, Claude in Chrome and computer use. Dynamic workflows and same-machine messaging are documented as working on API billing and on Amazon Bedrock. `/goal` works on cloud providers. Whether `/loop` needs a sign-in is unverified.
- Codex has hooks, sub-agents defined in TOML files, skills, `/goal`, `/compact` and `/memories`. It has no `/loop`. Its `/import` command reads a Claude Code or Cursor setup.
- Start with the defaults, and add pieces when you hit a problem they do not solve.

## Native commands

Checked against Claude Code v2.1.319. Several commands changed meaning in recent releases: `/review` was a separate pull-request review before v2.1.223, `/simplify` was the bug-finding command before v2.1.147, and `/agents` lost its interactive manager after v2.1.197.

### On the talk path

- `/goal <condition>` keeps the session working until a separate small model judges the condition met.
- `/loop` repeats a prompt on an interval; recurring tasks expire after 7 days.
- `/compact <focus>` summarises the session to free context, steered by the focus text.
- `/rc` (`/remote-control`) controls a session from another device.
- `/hooks`: Shows every configured hook and where it comes from, read-only. See [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).
- `/tasks`: Lists the session's background work, including sub-agents that have finished. See [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}).
- `/subtask`, `/fork`: `/subtask` starts a sub-agent that inherits the conversation; `/fork` copies the session into a separate background session. See [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}).
- `/list-agents`: Lists the sub-agents and other sessions this session can message. See [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}).
- `/workflows`: Watches, pauses and saves dynamic workflows, the scripts Claude writes to run many sub-agents. A workflow starts when you ask for one or use the `ultracode` keyword. See [Orchestration and workflows]({{ '/sections/10-orchestration/' | relative_url }}).
- `/context`: Shows what is filling the context window, by category, with suggestions. See [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}).
- `/code-review` (alias `/review`), `/security-review`: Reviews a branch or pull request in a separate context for bugs or security risks. See [Code review]({{ '/sections/13-code-review/' | relative_url }}).
- `/permissions` with auto mode: Sets allow, ask and deny rules; auto mode lets a classifier approve routine actions in long runs. See [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}).
- `/insights`: Writes a report on your recent sessions, friction points included, for improving your setup. See [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}).

### Also useful

- `/rewind`: Restores code, the conversation or both to a checkpoint.
- `/background` and agent view (`claude agents`): `/background` detaches the session to run as a background agent; agent view watches every background session.
- `/batch`: Splits one large change into 5 to 30 units, each run by a background sub-agent in its own worktree.
- `/simplify`: Runs four review agents in parallel for cleanup.
- `/verify`: Builds and runs the app and observes the change working.
- `/skill-doctor`: Shows what each skill costs in context and how often it is used.
- `/model` and `/effort`: Choose the model and the reasoning effort.
- `/advisor`: Lets a cheaper main model consult a stronger one at decision points.
- `/autocompact`: Sets how full the context window gets before automatic compaction.
- `/init` and `/memory`: `/init` writes a starter `CLAUDE.md`; `/memory` edits `CLAUDE.md` files and auto memory.
- `/clear`: Starts a new conversation with empty context.
- `/resume`: Returns to an earlier conversation.
- `/branch` and `/btw`: `/branch` forks the conversation in place to try another direction; `/btw` asks a side question without adding it to the conversation.
- `/schedule`: Creates cloud scheduled tasks, which run without the machine or an open session.
- `/plugin`: Manages plugins.
- `/reload-plugins`: Reloads plugins without a restart.
- `/reload-skills` and `/skills`: Manage skills without a restart.
- `/usage`: Shows session cost and usage attribution.
- `/doctor` and `/debug`: `/doctor` checks the setup and can audit instructions; `/debug` turns on debug logging for the session.
- `/fewer-permission-prompts` and `/update-config`: The first scans transcripts for common read-only calls and proposes an allowlist; the second edits `settings.json` from a plain request.
- `/plan` and `/sandbox`: Plan mode, and sandbox mode for Bash.
- `/deep-research`: The bundled research workflow.
- `/mcp` and `/statusline`: Manage MCP connections, and configure the status line.

### Availability on cloud providers

For Amazon Bedrock and similar cloud providers:

| Command or feature | On cloud providers |
|---|---|
| Dynamic workflows, `/workflows` | Available |
| `/goal` | Available; the evaluator runs on the session's provider |
| Auto mode | Available with supported models |
| Same-machine session messaging, `/list-agents` | Available from v2.1.248 |
| Cross-machine and cloud session messaging | Not available; needs Remote Control |
| `/remote-control` (`/rc`) | Not available, and API keys are not supported either |
| `/advisor` | Not available; Anthropic API only |
| `/code-review ultra` | Not available; falls back to a local review |
| `/insights` | Available |
| `/deep-research` | Needs the WebSearch tool; whether WebSearch exists on cloud providers is unverified |

Unverified: if you have a skill of your own named `review`, the docs do not say whether `/review` runs your skill or the built-in review.

Sources for this list, all verified 2026-11-09: <https://code.claude.com/docs/en/commands.md>, <https://code.claude.com/docs/en/workflows.md>, <https://code.claude.com/docs/en/goal.md>, <https://code.claude.com/docs/en/permission-modes.md>, <https://code.claude.com/docs/en/advisor.md>, <https://code.claude.com/docs/en/costs.md>, <https://code.claude.com/docs/en/code-review-local.md>, <https://code.claude.com/docs/en/cross-session-messaging.md>, <https://code.claude.com/docs/en/remote-control.md>.

## Examples from my own harness

- One install step puts the same context files, skills, hooks and agent profiles on every machine I use, so each new session starts with the same setup.

To start: one context file in one repository, with an install step added later.

## Pitfalls and lessons

- Turning on many features at once makes a failure hard to trace. Add one piece at a time.

## Sources

- `/goal`: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-08.
- `/loop`: <https://code.claude.com/docs/en/scheduled-tasks.md>, verified 2026-11-08.
- `/compact`: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.
- `/rc`, "API keys are not supported": <https://code.claude.com/docs/en/remote-control.md>, verified 2026-11-08.
- Codex commands, hooks, sub-agents and skills: <https://learn.chatgpt.com/docs/developer-commands.md?surface=cli>, <https://learn.chatgpt.com/docs/hooks.md>, <https://learn.chatgpt.com/docs/agent-configuration/subagents.md> and <https://learn.chatgpt.com/docs/build-skills.md>, verified 2026-11-08.
