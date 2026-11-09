---
title: "Detail"
parent: "Agentic AI fundamentals"
nav_order: 1
---

# Agentic AI fundamentals in detail

As of 2026-11-09.

## Purpose

Explain how a chat model becomes an agent, before any configuration.

## Key points

- A chat model answers one prompt. An agent works toward a goal over many steps.
- The agent loop: the model reads its context, chooses an action (usually a tool call), receives the result, and repeats until the task is done or it is stopped.
- Tools let it act: reading and editing files, running commands, searching, calling services.
- The context window is everything the model can see on one turn: instructions, the conversation and tool results. Several current Claude models support a context window of one million tokens.
- The model keeps no memory between sessions. Anything that must last is written to files the harness loads again.

## Examples from my own harness

None needed.

## Pitfalls and lessons

- Many agent failures trace back to context: the agent never saw the rule, or saw too much else.
- When an agent ignores an instruction, check whether the instruction was in its context at that moment.

## Sources

- Context window sizes: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.
- The agent loop as a general description: unverified against <https://code.claude.com/docs/en/overview.md>.
