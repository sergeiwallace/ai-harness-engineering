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

Sessions coordinate by sending each other messages. This page covers the kinds of channel, why one primary channel with ordered fallbacks beats several equal ones, how to probe a channel before using it, how to confirm delivery, and what each extra channel costs. The [section overview]({{ '/sections/09-session-coordination/' | relative_url }}) is the short version. A sub-topic page covers Claude Code's built-in [SendMessage and ListAgents]({{ '/sections/09-session-coordination/sendmessage-and-listagents.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Kinds of channel

| Channel | Reach | Latency | Good for | Weak at |
|---|---|---|---|---|
| Built-in messaging (`SendMessage`) | Sessions the tool can see: same machine, or others through Remote Control | Seconds; arrives in the target's conversation | Day-to-day routing | Sessions it cannot see; other tools |
| Agent mail service | Any agent connected to the service | Seconds to minutes; the agent must check its inbox | Mixed tools, durable threads | Needs the agent to poll |
| Message board | Anyone with access to the board | Minutes | Broadcasts, cross-machine notes | Point-to-point urgency |
| Git repository as inbox | Anyone who can pull the repo | Minutes; needs a pull | Machines with no other shared channel; a durable record | Speed |
| Typing into the session's terminal | One session on the same machine | Immediate | Last resort when the session is stuck | Easy to collide with a person typing; fragile |

Agent Mail (<https://github.com/Dicklesworthstone/mcp_agent_mail>) is an open-source mail service for coding agents, reached as an MCP server: agents get identities, inboxes and threads. Buzz (<https://github.com/block/buzz>) is an open-source project I use as a message board between machines.

## One primary, ordered fallbacks

A message sent through a channel that does not reach the right session leaves the sender waiting. With several equal channels, a sender has to guess which one the target reads, and the target has to check them all. With one primary and an ordered list of fallbacks, both sides know the rule: try the primary; if it cannot reach the target, try the next.

![A sequence diagram. The sender probes ListAgents through the native channel and learns the target is absent. In an optional fragment it probes the target's Agent Mail identity and learns it was last seen three hours ago. It commits the message to the git inbox; the target pulls its inbox and replies taken.]({{ '/assets/images/09-session-coordination/channel-ladder.svg' | relative_url }}){: style="width:100%"}

*Figure: walking the ladder: each probe asks whether the target will read it. Made with archify.*

### My order

| Where | Order |
|---|---|
| Same machine | `SendMessage`, then two fallbacks |
| Across machines | Native delivery, then Agent Mail, then Buzz, then the git inbox, then a last-resort channel |

- The order runs from fastest and most direct to slowest and most durable.
- A fallback is used when the step before it cannot reach the target. A slow reply on a working channel is a reason to wait, and the sender stays on that channel.

### Example: a cross-machine message

A chief on one machine needs to tell the chief on another that a shared library changed.

1. It runs `ListAgents`.
   - The other chief is not listed: the two machines are not linked through Remote Control.
2. It probes Agent Mail.
   - The other chief's identity exists and was active five minutes ago.
3. It sends there, asking for a reply.
4. The other chief replies within a few minutes.
   - Had the identity been stale, the sender would have gone on to the board, then the git inbox, as in the figure above.

## Probe before you send

Each channel can be up for you and still not reach the target. A probe answers "will this reach that session", not "is this channel up".

| Channel | Probe |
|---|---|
| Built-in messaging | Is the target in `ListAgents` output right now? |
| Agent mail | Does the target's identity exist, and when did it last check its inbox? |
| Board | Has the target posted or read recently? |
| Git inbox | Has the target's machine pulled the repo recently? |

A probe that only checks the channel (the mail server answers, the repo is reachable) still passes when the target is not listening.

## Confirm delivery

A successful send tells you the sender handed the message over. Count a message as delivered only when the target replies or acknowledges it.

| Signal | Means |
|---|---|
| The send call returned success | The channel accepted it |
| The send call timed out | Unknown: it may have arrived |
| The target replied "taken" | Delivered and read |
| No reply after the expected time | Escalate to the next channel or the coordinator |

### Example: the timeout that delivered

1. A sender's `SendMessage` call times out.
2. The sender assumes it failed and sends the same task through the fallback.
3. The first message had in fact arrived.
   - The target now has the task twice, and starts it twice.

Treat a timeout as unknown. Ask the target "did you get task X" before re-sending, and give each task an id so a duplicate is visible.

### Acknowledgements

For anything that matters, ask for an explicit acknowledgement in the message.

```text
Task export-01 ... Reply "taken export-01" when you start.
```

- Put the task id in the acknowledgement, so a reply matches its task.
- Set a time after which a missing acknowledgement triggers the next channel.

## What each extra channel costs

Each extra channel adds its own probes, fallbacks and failures. A channel is a small system of its own:

- Setup on every machine that uses it.
- Credentials, and their renewal.
- A probe for whether the target reads it.
- A way for each session to read it.
- A test that it still works.

A channel nobody uses for weeks breaks without anyone noticing, and fails the one time it is needed. Before adding one, check that it reaches something the existing ones cannot.

## Agent teams

Agent teams let agents message each other directly.

- They are experimental and turned on with an environment flag.
- A teammate cannot start a team of its own.
- They suit a group of agents working one problem inside one session's lifetime.
- Long-running sessions across machines still need the channels above.

## Examples from my own harness

- A fixed order of message channels: `SendMessage` first, then Agent Mail, then Buzz, then a git-based inbox for other machines. Each channel is checked for reachability before use.
- A last-resort channel that types a prompt into a stuck session's terminal, used only when nothing else reaches it, and never into a terminal a person is typing in.
- Replies addressed by the sender id on the incoming message, because replying to a stale id can wake a session that had already finished.

To start: use `SendMessage` alone, and ask for a reply on every message that matters.

## Pitfalls and lessons

- Pick one primary channel and make the rest fallbacks.
- Treat a timeout as unknown. Ask the target before re-sending.
- A channel that is never exercised stops working unnoticed. Test each fallback on a schedule, or drop it.

## Sources

- `ListAgents` and `SendMessage`: <https://code.claude.com/docs/en/cross-session-messaging.md>, verified 2026-11-08.
- Remote Control requirements: <https://code.claude.com/docs/en/remote-control.md>, verified 2026-11-08.
- Agent Mail: <https://github.com/Dicklesworthstone/mcp_agent_mail>. Buzz: <https://github.com/block/buzz>.

{% include page_nav.html %}
