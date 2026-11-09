---
title: "SendMessage and ListAgents"
parent: "Message channels and fallbacks"
grand_parent: "Coordinating several sessions"
nav_order: 1
---

{% include page_nav.html %}

# SendMessage and ListAgents

As of 2026-11-09.

A sub-topic of [message channels and fallbacks]({{ '/sections/09-session-coordination/message-channels.html' | relative_url }}). It covers Claude Code's built-in cross-session messaging: what the two tools do, what they reach, how a message arrives, and patterns for using them well.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The two tools

In Claude Code, the `ListAgents` tool lists running agents and sessions, and `SendMessage` sends one a message. `/list-agents` is the slash command a person types to see the same list.

![A sequence diagram. Session A calls ListAgents through Claude Code, which lists session B as idle. A sends a task; Claude Code delivers it to B as a new turn. B replies taken, then later reports done.]({{ '/assets/images/09-session-coordination/sendmessage-exchange.svg' | relative_url }}){: style="width:100%"}

*Figure: a SendMessage exchange; delivery is confirmed by the reply. Made with archify.*

| Tool | Input | Output |
|---|---|---|
| `ListAgents` | None | The sub-agents and sessions this session can message, with their state |
| `SendMessage` | A target from that list, and the message text | The message is delivered into the target's conversation |

## Reach

| Where the target is | Reachable with `SendMessage`? |
|---|---|
| Same machine, any provider (API billing included) | Yes |
| Another machine, with Remote Control | Yes; Remote Control needs a claude.ai subscription |
| Another machine, without Remote Control | No: it is not in the list at all |
| A Codex session on the same machine | No: it is a different tool |

A cross-machine setup without Remote Control needs the fallbacks on the [parent page]({{ '/sections/09-session-coordination/message-channels.html' | relative_url }}).

### Example: what the list shows and what it does not

```text
> ListAgents
  docs-writer       idle
  test-migrator     running
  reviewer (sub-agent of this session)   finished
```

A Codex session on the same machine is not in the list: it is a different tool. A Claude Code session on another machine without Remote Control is not in the list either. Absence from the list means "this tool cannot reach it", not "it is not running".

## Writing a good message

A message lands in the middle of the target's work, as a new turn. It should stand on its own.

- Say who is asking and why, in one line.
- State the task or the question, and the evidence wanted back.
- Give the task an id the reply can quote.
- Ask for an acknowledgement when it matters.

### Example: before and after

Before:

```text
can you look at the export thing
```

After:

```text
From the chief, task export-01: the nightly export failed at 02:10 with a
timeout. Find the cause and fix it with a regression test. Reply "taken
export-01" when you start, and the commit plus test output when done.
```

The second needs no follow-up question, and its reply can be matched to the task.

## Patterns

### Check the list before each send

1. Run `ListAgents`.
2. If the target is listed, send.
3. If it is missing, go to the fallback, or start a new session for the task.

Sessions come and go, and a target that was listed an hour ago may have exited.

### Reply to the sender id on the message

- Use the sender id carried on the incoming message when replying.
- Do not reuse an id remembered from earlier.
  - Ids can change when a session restarts.
  - An old id can point at a finished agent, and the reply then resumes it.

### One task per message

| One message, three tasks | Three messages, three ids |
|---|---|
| One reply mixes three answers | One reply per task |
| One task gets forgotten | Each task has its own acknowledgement |
| Hard to track | Each id is a row in the tracker |

## Agent teams compared

| | `SendMessage` between sessions | Agent teams |
|---|---|---|
| Who | Sessions that already exist and live independently | A group started together for one job |
| Lifetime | Hours or days | One session's lifetime |
| Status | Built in | Experimental, turned on with an environment flag |
| Nesting | Any session can message any listed one | A teammate cannot start a team of its own |

## Sources

- `ListAgents` and `SendMessage`: <https://code.claude.com/docs/en/cross-session-messaging.md>, verified 2026-11-08.
- Remote Control requirements: <https://code.claude.com/docs/en/remote-control.md>, verified 2026-11-08.

{% include page_nav.html %}
