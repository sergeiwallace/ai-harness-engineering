---
title: "Agentic AI fundamentals"
nav_order: 11
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Agentic AI fundamentals

As of 2026-11-09.

Stop 2 on the talk path, about 2 minutes. This page is an overview you can learn from on its own. [Agentic AI fundamentals in detail]({{ '/sections/01-agentic-ai-fundamentals/detail.html' | relative_url }}) goes deeper on each idea.

## From a chat model to an agent

A chat model answers one prompt. You type a question, it writes a reply, and the exchange ends there. If the reply needs a file read, a command run or a web page checked, you do that yourself and paste the result back.

An agent works toward a goal over many steps. You give it a task, such as "fix the failing test in this repository", and it decides what to look at, does it, reads what came back, and decides what to do next. It keeps going until the task is done, it gets stuck, or you stop it. The model underneath is the same kind of model as in a chat app. What changes is the software around it, which lets the model act and feeds the results of its actions back in.

## The agent loop

Every coding agent, Claude Code and Codex included, runs the same basic loop.

1. Read the context. The model sees its instructions, the task, the conversation so far and the results of earlier actions.
2. Choose an action. Usually that is a tool call: read a file, search, edit, run a command.
3. Run the tool. The client program on your machine runs the tool for the model. The model only asks.
4. Add the result. The tool's output goes back into the context, and the loop starts again.

The loop ends when the model decides the task is done and answers without a tool call, or when something outside it stops the run: you, a time or cost limit, or a check that refuses to let it finish.

The split in step 3 matters for everything later on this site. The model never touches your files directly. It writes a request, and the client decides whether to run it. That is where permissions, approval prompts and the checks called hooks attach. See [deep dive: the agent loop]({{ '/sections/01-agentic-ai-fundamentals/detail.html' | relative_url }}#the-agent-loop).

## Tools

Tools are what let an agent act. A coding agent ships with tools to read, search and edit files, run shell commands and fetch web pages, and you can add more. Each tool has a name, a description and the inputs it takes. The model reads those descriptions in its context and picks a tool by name, much as a person picks a command from a menu.

More tools are not always better. Every tool description takes up room in the context on every turn, and a model with many similar tools picks the wrong one more often. See [deep dive: tools]({{ '/sections/01-agentic-ai-fundamentals/detail.html' | relative_url }}#tools).

## The context window

The context window is everything the model can see on one turn: its instructions, the conversation, every tool result so far. It has a fixed size, measured in tokens (roughly three quarters of a word each). Several current Claude models support a window of one million tokens, which is a lot, but an agent that reads large files and long command outputs fills it faster than you would expect.

Two facts about the window drive most of what a harness does.

- The model knows only what is in the window. If a rule is not in the context at the moment the agent acts, the agent cannot follow it.
- Nothing carries over between sessions on its own. When a session ends, or its history is summarised to free space, everything not written to a file is gone.

See [deep dive: the context window]({{ '/sections/01-agentic-ai-fundamentals/detail.html' | relative_url }}#the-context-window) and [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}).

## A worked example

Suppose you ask an agent to find out why a test fails and fix it. A typical run looks like this.

1. The agent runs the test suite with its shell tool. The output, with the failure message, goes into the context.
2. It searches the code for the function named in the failure and reads that file.
3. It edits the function, then runs the one failing test again.
4. The test passes. It runs the whole suite to check nothing else broke, and reports what it changed.

That is four turns of the loop, each one a tool call whose result shaped the next choice. Now suppose your project has a rule that every change needs a new regression test. If that rule is written in a file the agent loaded at the start, it will usually follow it. If the rule lives only in your head, or in a document the agent never opened, it cannot. Most agent mistakes look like this: the agent did something reasonable with the context it had, and the context was missing something.

## Why this matters for the rest of the site

A harness is the set of things you put around this loop. Context files decide what is in the window at the start. Tools and skills decide what the agent can do. Hooks and checks run at step 3, where the client handles each tool call, and can refuse an action. Memory files carry state across sessions. Each later page takes one of these parts in turn, starting with [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}).

## Pages for this topic

- [Agentic AI fundamentals in detail]({{ '/sections/01-agentic-ai-fundamentals/detail.html' | relative_url }}): the loop step by step, tool calling, the context window and tokens, sessions and state, stopping and control, and the common failure modes.

{% include page_nav.html %}
