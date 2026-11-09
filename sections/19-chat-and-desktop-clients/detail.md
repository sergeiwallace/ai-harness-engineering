---
title: "Detail"
parent: "Chat, desktop and web clients"
nav_order: 1
---

{% include page_nav.html %}

# Chat and desktop clients in detail

As of 2026-11-09.

## Purpose

Say what carries over from harness engineering to the chat, desktop and web clients, what applies in adapted form, and what does not apply.

## Key points

- Carries over: the Claude Code desktop app and the web version run "the same underlying engine", and configuration, project memory and MCP servers are shared across the local surfaces.
- Applies in adapted form: in the claude.ai chat app, project instructions and uploaded files play the role of context files (unverified; check the claude.ai help pages). The same plugin format installs on claude.ai and in Cowork, where a different set of components loads.
- Does not apply: hooks, scripts and local guards need a local client. A cloud session does not load the plugins in your local settings.
- The habits carry over everywhere: write durable notes, keep the original request, check results, and decide what needs your approval.

## Examples from my own harness

None; my harness runs in the terminal clients.

## Pitfalls and lessons

- In a chat app every rule is an instruction, so expect it to be followed less consistently than a rule a hook enforces.

## Sources

- Surfaces and what they share: <https://code.claude.com/docs/en/platforms.md>, verified 2026-11-08.
- Plugins on claude.ai and cloud sessions: <https://code.claude.com/docs/en/plugins/overview.md>, verified 2026-11-08.

{% include page_nav.html %}
