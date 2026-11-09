---
title: "Detail"
parent: "Browser automation and computer use"
nav_order: 1
---

{% include page_nav.html %}

# Browser automation in detail

As of 2026-11-09.

## Purpose

Cover how agents use a browser or a computer, and how to use screenshots as evidence.

## Key points

- Claude in Chrome connects Claude Code to a Chrome extension. It needs a direct Anthropic plan and is not available through providers such as Amazon Bedrock.
- Computer use in the Claude Code CLI is a research preview on macOS for Pro, Max and Ultra plans.
- Driving a browser yourself through the Chrome DevTools Protocol (CDP) or Playwright works with any provider.
- Screenshots let an agent compare what it built with what was intended.
- Browser automation is slow and breaks easily. Use an API or a file when one exists.
- A web page can contain instructions aimed at the agent. This is prompt injection in the security sense. Treat page content as untrusted.

## Examples from my own harness

- A CDP automation skill that drives a dedicated browser profile in the background, so it never takes over my screen, with a list of non-browser routes to try first.
- A screenshot skill for attaching evidence.

To start: ask the agent to take a screenshot of what it changed.

## Pitfalls and lessons

- An agent that takes over the screen interrupts the person using it. Run browser automation in the background, in its own profile.

## Sources

- Claude in Chrome: <https://code.claude.com/docs/en/chrome.md>, verified 2026-11-08.
- Computer use: <https://code.claude.com/docs/en/computer-use.md>, verified 2026-11-08.

{% include page_nav.html %}
