---
title: "Detail"
parent: "Coordinating several sessions"
nav_order: 1
---

{% include page_nav.html %}

# Session coordination in detail

As of 2026-11-09.

## Purpose

Show how several long-running sessions and agents coordinate, and how one person keeps track of them.

## Key points

- Running several sessions at once multiplies output. It also multiplies context switching for the person.
- A coordinator session takes requests from the person, hands them to other sessions or agents, and reports back.
- In Claude Code, the `ListAgents` tool lists running agents and sessions, and `SendMessage` sends one a message. On the same machine this works with any provider, including API billing. Reaching another machine needs Remote Control, which needs a claude.ai subscription.
- Separate messaging systems fill the gaps: a shared mail service for agents, a message board, or a git repository used as an inbox.
- One session per machine can own shared actions such as installs and pushes, so two sessions never do them at once.
- `/list-agents`: Lists the sub-agents and other sessions this session can message.

## Examples from my own harness

- A coordination layer built on firstmate, an open-source project by GitHub user kunchenguid (<https://github.com/kunchenguid/firstmate>). I use my own public fork with customizations for my setup (<https://github.com/sergeiwallace/firstmate>). In it, one firstmate session takes my requests and supervises a chief-of-staff session on each machine. A chief-of-staff session routes work to the other sessions on its machine and runs pushes and merges one at a time.
- A fixed order of message channels: `SendMessage` first, then Agent Mail (<https://github.com/Dicklesworthstone/mcp_agent_mail>), then Buzz (<https://github.com/block/buzz>), then a git-based inbox for other machines. Each channel is checked for reachability before use.

To start: two sessions and one shared notes file.

## Pitfalls and lessons

- A message sent through a channel that does not reach the right session leaves the sender waiting. Pick one primary channel and make the rest fallbacks.
- Each extra channel adds its own probes, fallbacks and failures.

![You send requests to one coordinator session, which uses SendMessage to hand work to three worker sessions that report back. When SendMessage cannot reach a session, the coordinator falls back to Agent Mail, Buzz or a git inbox.]({{ '/assets/diagrams/coordination.svg' | relative_url }})

## Sources

- firstmate: <https://github.com/kunchenguid/firstmate>, verified 2026-11-08. My fork: <https://github.com/sergeiwallace/firstmate>.
- `ListAgents` and `SendMessage`: <https://code.claude.com/docs/en/cross-session-messaging.md>, verified 2026-11-08.
- Remote Control requirements: <https://code.claude.com/docs/en/remote-control.md>, verified 2026-11-08.

{% include page_nav.html %}
