---
title: "Message channels and fallbacks"
parent: "Coordinating several sessions"
nav_order: 2
has_children: true
has_toc: false
redirect_from:
  - /sections/09-session-coordination/advanced.html
---

{% include page_nav.html %}

# Message channels and fallbacks

As of 2026-11-09.

Sessions coordinate by sending each other messages, and no single channel reaches every session. This page covers the kinds of channel, how to order them as one primary with fallbacks, how to probe a target before sending, how to confirm delivery, the last-resort channel, and what each extra channel costs. The [section overview]({{ '/sections/09-session-coordination/' | relative_url }}) is the short version; the sub-topic page covers the built-in [SendMessage and ListAgents]({{ '/sections/09-session-coordination/sendmessage-and-listagents.html' | relative_url }}) in depth.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Kinds of channel

| Channel | Reach | How fast | Good for | Weak at |
|---|---|---|---|---|
| Built-in messaging (`SendMessage`) | Your sessions on this machine; with Remote Control, your sessions on other machines and in the cloud | Seconds; arrives in the target's conversation between tool calls | Day-to-day routing | Other tools; machines without Remote Control |
| An agent mail service | Any agent connected to the service, whatever its tool | Seconds to minutes; the agent checks its inbox | Mixed tools, durable threads | Needs the agent to check |
| A message board | Anyone with access to the board | Minutes | Broadcasts, cross-machine notes | Point-to-point urgency |
| A git repository as inbox | Anyone who can pull the repository | Minutes; needs a pull | Machines with no other shared channel; a durable record | Speed |
| Channels (Claude Code, research preview) | Events from outside systems into one running session | Immediate | CI results, chat bridges, webhooks | Session-to-session messages |
| Typing into the session's terminal | One session on the same machine | Immediate | Last resort, and slash commands | Collisions with a person typing; fragile |

### The tools behind them

| Tool | What it is | Notes |
|---|---|---|
| Agent Mail (<https://github.com/Dicklesworthstone/mcp_agent_mail>) | A "mail-like coordination layer for coding agents", reached as an HTTP MCP server | Identities, inbox and outbox, searchable threads, advisory file reservations, a web UI for people; the Python version points to a Rust rewrite |
| Buzz (<https://github.com/block/buzz>) | "A hive mind communication platform": a self-hostable workspace where people and agents share channels | Every message and event is a signed entry in one log; Apache 2.0 |
| Claude Code channels | MCP servers that push events into a running session, started with `claude --channels plugin:<name>@<marketplace>` | Telegram, Discord and iMessage plugins ship with the preview; not available on Bedrock and the other cloud providers |

## One primary, ordered fallbacks

A message sent through a channel the target does not read leaves the sender waiting. With several equal channels, the sender has to guess and the target has to check them all. With one primary and an ordered list of fallbacks, both sides follow the same rule:

1. Try the primary channel.
2. If it cannot reach the target, try the next one.
3. Stop at the first channel that reaches the target.

![A sequence diagram. The sender probes ListAgents through the native channel and learns the target is absent. In an optional fragment it probes the target's Agent Mail identity and learns it was last seen three hours ago. It commits the message to the git inbox; the target pulls its inbox and replies taken.]({{ '/assets/images/09-session-coordination/channel-ladder.svg' | relative_url }}){: style="width:100%"}

*Figure: walking the ladder: each probe asks whether the target will read it. Made with archify.*

### My order

| Where | Order |
|---|---|
| Same machine | `SendMessage`, then two fallbacks |
| Across machines | Native delivery, then Agent Mail, then Buzz, then the git inbox, then a last-resort channel |

- The order runs from fastest and most direct to slowest and most durable.
- A fallback is for a target the step before it cannot reach. A slow reply on a working channel is a reason to wait on that channel.

### Example: a cross-machine message

A chief on one machine needs to tell the chief on another that a shared library changed.

1. It runs `ListAgents`.
   - The other chief is not listed: the two machines are not linked through Remote Control.
2. It probes Agent Mail.
   - The other chief's identity exists and was active five minutes ago.
3. It sends there, asking for a reply.
4. The other chief replies within a few minutes.
   - Had the identity been stale, the sender would have gone on to the board, then the git inbox.

## Probe before you send

A channel can be up for you and still not reach the target. A probe asks whether the target will read the message.

| Channel | Probe | Reads as unreachable when |
|---|---|---|
| Built-in messaging | Is the target in the `ListAgents` output now? | Missing, or marked `can't receive cross-session messages` |
| Agent mail | Does the target's identity exist, and when did it last check its inbox? | No identity, or last seen hours ago |
| Board | Has the target posted or read recently? | No activity in the expected window |
| Git inbox | Has the target's machine pulled recently? | No pull since the last message |

- A probe that only checks the channel (the mail server answers, the repository is reachable) still passes when the target is not listening.
- A built-in row marked `offline` is a special case: the message goes through and arrives when that machine reconnects.

## Confirm delivery

A successful send says the channel accepted the message. Count a message as delivered only when the target replies or acknowledges it.

| Signal | Means |
|---|---|
| The send returned success | The channel accepted it; on the built-in channel, the receiver may still hold it for approval or refuse it |
| The send result begins `Not sent` | Refused before it left: the target cannot receive messages |
| The send timed out | Unknown: it may have arrived |
| The target replied "taken" | Delivered and read |
| No reply after the expected time | Escalate to the next channel or the coordinator |

### Example: the timeout that delivered

1. A sender's `SendMessage` call times out.
2. The sender assumes it failed and sends the same task through the fallback.
3. The first message had in fact arrived.
   - The target now has the task twice, and starts it twice.

What prevents it:

- Treat a timeout as unknown, and ask the target "did you get task X" before re-sending.
- Give each task an id, so a duplicate is visible to the target.
- In a tool or hook, refuse a second send of the same task id within a set window; the check then holds even when the sender forgets.

### Acknowledgements

For anything that matters, ask for an explicit acknowledgement in the message.

```text
Task export-01 ... Reply "taken export-01" when you start.
```

- Put the task id in the acknowledgement, so a reply matches its task.
- Set a time after which a missing acknowledgement triggers the next channel.

## The last-resort channel

Typing into a session's terminal reaches a session that no other channel reaches, and it is the only way to send a slash command: a `SendMessage` message that contains `/compact` arrives as plain text and does not run.

| Use it for | Guard it with |
|---|---|
| A stuck session that reads no inbox | A check that no person is typing in that pane |
| A slash command, such as `/compact` from a [context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }}) | Targeting by session name, never by pane position |
| | A check afterwards that the command took effect |

Put those guards in the library that does the typing, so every caller gets them.

## What each extra channel costs

Each extra channel adds its own probes, fallbacks and failures. A channel is a small system of its own:

- Setup on every machine that uses it.
- Credentials, and their renewal.
- A probe for whether the target reads it.
- A way for each session to read it.
- A test that it still works.

A channel nobody uses for weeks breaks without anyone noticing. Before adding one, check that it reaches something the existing ones cannot, and add a scheduled test that sends through it.

## Agent teams

Agent teams are a different shape of multi-agent work:

- A lead session spawns teammates, which message each other through mailboxes and share a task list.
- Everything lives inside the lead session's lifetime.
- They are experimental and turned on with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.

Long-running sessions across machines still need the channels on this page. The comparison is on [SendMessage and ListAgents]({{ '/sections/09-session-coordination/sendmessage-and-listagents.html' | relative_url }}#agent-teams).

## Examples from my own harness

- A fixed order of message channels: `SendMessage` first, then Agent Mail, then Buzz, then a git-based inbox for other machines.
  - Each channel is probed for the target before use.
- A last-resort channel that types a prompt into a stuck session's terminal.
  - It refuses a terminal a person is typing in.
- Replies addressed to the sender on the incoming message.
  - Replying to an id remembered from earlier once woke a session that had already finished.

To start: use `SendMessage` alone, and ask for a reply on every message that matters.

## Pitfalls and lessons

| Pitfall | What helps |
|---|---|
| Several equal channels | One primary channel, ordered fallbacks |
| A timeout read as a failure | Ask before re-sending; refuse duplicate task ids in the tool |
| A probe that checks the channel | Probe whether the target reads it |
| A fallback nobody exercises | A scheduled test send, or drop the channel |
| A slash command sent as a message | Type it into the terminal through a guarded library |

## Sources

- Cross-session messaging: <https://code.claude.com/docs/en/cross-session-messaging.md>, verified 2026-11-09.
- Channels: <https://code.claude.com/docs/en/channels.md>, verified 2026-11-09.
- Agent teams: <https://code.claude.com/docs/en/agent-teams.md>, verified 2026-11-09.
- Agent Mail: <https://github.com/Dicklesworthstone/mcp_agent_mail>, verified 2026-11-09. Buzz: <https://github.com/block/buzz>, verified 2026-11-09.

{% include page_nav.html %}
