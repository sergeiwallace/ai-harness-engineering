---
title: "Native commands"
parent: "Claude Code and Codex basics"
nav_order: 2
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Native commands

As of 2026-11-09, checked against Claude Code v2.1.319. Part of [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}). Slide: [stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

A native command is a slash command that ships with the client, typed in the session, such as `/compact`. This page groups Claude Code's commands by what they are for, says when each one is the right tool, and ends with a worked session that uses several together. The two commands that keep a session working by themselves have a page of their own: [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The map

The commands fall into the phases of a task. The diagram on the [overview]({{ '/sections/03-claude-code-and-codex/' | relative_url }}) shows those phases in one session; this table maps each phase to the commands that act on it.

| Phase | Question you are asking | Commands |
|---|---|---|
| Before you start | Is there room, and is the setup right? | `/context`, `/hooks`, `/permissions`, `/model`, `/effort` |
| Doing the work | How do I keep it going, or split it? | `/goal`, `/loop`, `/batch`, `/subtask`, `/fork`, `/background` |
| While it runs | What is happening? | `/tasks`, `/workflows`, `/list-agents` |
| Checking the result | Is it right? | `/code-review`, `/security-review`, `/simplify`, `/verify` |
| Keeping the session healthy | Can I keep going in this window? | `/compact`, `/clear`, `/resume`, `/rewind`, `/branch` |
| Improving the setup | What keeps going wrong? | `/insights`, `/fewer-permission-prompts`, `/doctor`, `/skill-doctor` |

> Several commands changed meaning in recent releases: `/review` was a separate pull-request review before v2.1.223, `/simplify` was the bug-finding command before v2.1.147, and `/agents` lost its interactive manager after v2.1.197. Check the reference for your installed version before you script against a command.

## Running long work

### The commands

| Command | What it does | Detail |
|---|---|---|
| `/goal <condition>` | Keeps the session working until a small evaluator model judges the condition met | [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}) |
| `/loop` | Repeats a prompt on an interval or at a pace the agent sets; recurring tasks expire after 7 days | [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}#loop) |
| `/schedule` | Creates cloud scheduled tasks that run without the machine or an open session | |
| `/workflows` | Watches, pauses and saves dynamic workflows, the scripts Claude writes to run many sub-agents | [Orchestration]({{ '/sections/10-orchestration/' | relative_url }}) |

### Choosing between them

| You want | Use |
|---|---|
| One task finished, however many turns it takes | `/goal` |
| The same check repeated while you are in the session | `/loop` |
| A job that runs even with your laptop closed | `/schedule` |
| Many sub-agents on parts of one big task | a dynamic workflow, watched with `/workflows` |

A workflow starts when you ask for one or use the `ultracode` keyword.

## Splitting and tracking work

### The commands

- `/tasks` lists the session's background work, including sub-agents that have finished.
- `/subtask` starts a sub-agent that inherits the conversation; `/fork` copies the session into a separate background session.
- `/background` detaches the session to run as a background agent; agent view (`claude agents`) watches every background session.
- `/batch` splits one large change into 5 to 30 units, each run by a background sub-agent in its own worktree.
- `/list-agents` lists the sub-agents and other sessions this session can message.

### A rename across a large codebase

![Sequence: you ask /batch to rename a key one unit per package; the session plans 12 units and starts sub-agents in their own worktrees; one reports done, another fails on a fixture; /tasks shows 11 done and 1 failed, and you send the fix for that unit]({{ '/assets/images/03-claude-code-and-codex/batch-sequence.svg' | relative_url }})

*A rename across packages with `/batch`. Each unit runs in its own worktree, so a failure stays in one package. Diagram made with archify; the task is an illustration.*

```text
> /batch rename the config key `timeout_ms` to `request_timeout_ms` everywhere,
  update the tests, one unit per package

  ⎿  Planned 12 units. Each runs in its own worktree.
> /tasks
  ⎿  9 done · 2 running · 1 failed (packages/billing: test fixture hard-codes the old key)
```

Each unit has its own worktree, so units do not collide on the same files, and the one failure is isolated to one package. The excerpt is an illustration. Delegation and coordination have their own sections: [Delegation]({{ '/sections/08-delegation/' | relative_url }}) and [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}).

## Context and session state

### The commands

| Command | What it does |
|---|---|
| `/context` | Shows what fills the context window, by category, with suggestions |
| `/compact <focus>` | Summarises the session to free context, steered by the focus text |
| `/autocompact` | Sets how full the window gets before automatic compaction |
| `/clear` | Starts a new conversation with empty context |
| `/resume` | Returns to an earlier conversation |
| `/rewind` | Restores code, the conversation or both to a checkpoint |
| `/branch` | Forks the conversation in place to try another direction |
| `/btw` | Asks a side question without adding it to the conversation |
| `/init` | Writes a starter `CLAUDE.md` from the repository |
| `/memory` | Edits `CLAUDE.md` files and auto memory |

### What /rewind does not undo

`/rewind` tracks file edits made through the client's edit tools. Files changed only by shell commands are not tracked, and sub-agent edits are usually not restored. Keep committing to git; a checkpoint does not replace it.

### Compact with a focus

`/compact` without a focus keeps what the summariser thinks matters. With a focus it keeps what you name:

```text
> /compact keep the root cause, the failing test name and the commit id
```

[Context window and memory]({{ '/sections/11-context-window/' | relative_url }}) covers compaction in depth.

## Configuration and enforcement

### The commands

- `/hooks` shows every configured hook and where it comes from, read-only.
- `/permissions` sets allow, ask and deny rules; auto mode lets a classifier approve routine actions in long runs.
- `/plan` and `/sandbox` switch to plan mode and to sandbox mode for shell commands.
- `/fewer-permission-prompts` scans transcripts for common read-only calls and proposes an allowlist.
- `/update-config` edits `settings.json` from a plain request.
- `/model` and `/effort` choose the model and the reasoning effort; `/advisor` lets a cheaper main model consult a stronger one at decision points.
- `/plugin`, `/reload-plugins`, `/reload-skills` and `/skills` manage plugins and skills without a restart; `/skill-doctor` shows what each skill costs in context and how often it is used.
- `/mcp` manages MCP connections; `/statusline` configures the status line.

### From prompt fatigue to an allowlist

```text
> /fewer-permission-prompts
  ⎿  Proposed allow rules from your recent sessions:
       Bash(git status)        Bash(git diff *)        Bash(pytest *)
     Add to .claude/settings.local.json? (y/n)
```

The rules land in the local file, so they apply to you in this project only. [Configuration and permissions]({{ '/sections/03-claude-code-and-codex/configuration-and-permissions.html' | relative_url }}) explains the rule syntax and the precedence.

## Review and verification

| Command | What it does | See |
|---|---|---|
| `/code-review` (alias `/review`) | Reviews a branch, pull request, path or commit range in a separate context for bugs | [Code review]({{ '/sections/13-code-review/' | relative_url }}) |
| `/security-review` | Checks the diff for security risks | |
| `/simplify` | Runs four review agents in parallel for cleanup | |
| `/verify` | Builds and runs the app and observes the change working | [Testing]({{ '/sections/12-testing/' | relative_url }}) |
| `/deep-research` | The bundled research workflow | |

## Measuring use

- `/insights` writes a report on your recent sessions, friction points included, for improving your setup. See [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}).
- `/usage` shows session cost and usage attribution.
- `/doctor` checks the setup and can audit instructions; `/debug` turns on debug logging for the session.

## Remote control

`/rc` (`/remote-control`) controls a session from another device. It needs a claude.ai sign-in; API keys are not supported, and it is not available through cloud providers.

## A morning's session

```text
> /context
  ⎿  System prompt and tools: 9%   Memory files: 4%   Messages: 12%   Free: 75%

> /goal the flaky test in tests/test_sync.py passes 20 runs in a row
  and the fix is committed

● Bash(for i in $(seq 20); do python -m pytest tests/test_sync.py -q || break; done)
  ⎿  ... 3 passed, then 1 failed: timeout waiting for lock

  ... (the agent finds a missing release in a cleanup path, fixes it,
       reruns 20 times, commits)

  ⎿  Goal met: 20 consecutive passes shown, commit a1b2c3d created.

> /code-review HEAD~1..HEAD
  ⎿  No correctness findings.

> /compact keep the root cause and the commit id
```

| Step | Command | Why it is there |
|---|---|---|
| 1 | `/context` | Confirms there is room before a long task |
| 2 | `/goal` | Holds the session to a checkable condition |
| 3 | `/code-review` | Gets a second read in a separate context |
| 4 | `/compact` | Frees space and keeps the two facts the next task needs |

The percentages and output are illustrative.

## Codex commands

Codex has its own set. The ones that match Claude Code's are `/goal`, `/compact` and `/memories`; it has no `/loop`, and it adds `/import` to read a Claude Code or Cursor setup. See [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}).

## Sources

- <https://code.claude.com/docs/en/commands.md>, <https://code.claude.com/docs/en/workflows.md>, <https://code.claude.com/docs/en/goal.md>, <https://code.claude.com/docs/en/permission-modes.md>, <https://code.claude.com/docs/en/advisor.md>, <https://code.claude.com/docs/en/costs.md>, <https://code.claude.com/docs/en/code-review-local.md>, <https://code.claude.com/docs/en/cross-session-messaging.md>, <https://code.claude.com/docs/en/remote-control.md>, verified 2026-11-09.
- `/rewind`: <https://code.claude.com/docs/en/checkpointing.md>, verified 2026-11-09.
- Codex: <https://learn.chatgpt.com/docs/developer-commands.md?surface=cli>, verified 2026-11-08.
- Session excerpts are illustrations written for this page.

{% include page_nav.html %}
