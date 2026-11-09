---
title: "Chat, desktop and web clients"
nav_order: 29
has_children: true
has_toc: false
permalink: /sections/19-chat-and-desktop-clients/
---

{% include page_nav.html %}

# Chat, desktop and web clients

As of 2026-11-09. Overview page. This section has no stop of its own on the talk path; [Stop 1]({{ '/talk/' | relative_url }}#stop-1-2-minutes) says in one line that most of the ideas carry over.

Most of this site is about coding agents in a terminal: Claude Code and Codex. Many people use the same models through a chat app, a desktop app or a browser. The question for them is which parts of harness engineering still apply. The short answer: the Claude Code desktop and web versions keep almost everything, a chat app keeps the habits and an adapted form of context, and anything that depends on running code on your machine, such as hooks, does not carry over to a chat app at all.

![Sequence diagram comparing the same review checklist in a chat project and in Claude Code. In the chat project the checklist sits in the project instructions, the model reads them and returns a review that skips one item, and you have to check the list yourself. In Claude Code the checklist is a skill, and when the agent tries to finish, a hook blocks it because item 4 is missing, so the review comes back with every item covered.]({{ '/assets/images/19-chat-and-desktop-clients/chat-vs-cli-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. The same checklist as an instruction in a chat project and as a skill plus a hook in Claude Code. Diagram made with archify.*

## The three tiers

| Tier | What it covers | Examples |
|---|---|---|
| Carries over | Almost the whole harness | Claude Code desktop app, Claude Code on the web |
| Applies in adapted form | Context, skills and habits, as instructions | The claude.ai chat app with projects; Cowork |
| Does not apply | Anything that runs code on your machine | Hooks, local scripts, local permission rules |

## 1. What carries over: Claude Code desktop and web

### 1.1 Same engine, different window

The Claude Code desktop app and the web version run "the same underlying engine" as the terminal client. Configuration, project memory and MCP servers are shared across the local surfaces, so a `CLAUDE.md`, a skill or an MCP server you set up for the terminal is there in the desktop app too.

### 1.2 What to check on the web

A web session runs in the cloud. A cloud session does not load the plugins in your local settings. Anything that lives only in your local configuration needs to be in the repository, or set up for the cloud environment, before a web session can use it.

| Part | Terminal | Desktop app | Web (cloud session) |
|---|---|---|---|
| `CLAUDE.md` in the repository | Yes | Yes | Yes |
| Skills and MCP servers in local settings | Yes | Yes (shared) | Not from your local settings |
| Plugins in your local settings | Yes | Yes | No |
| Hooks that call scripts on your machine | Yes | Yes | No; your machine is not part of the session |

> Note: the web column is based on the plugin and platform docs listed under Sources. Check your surface's documentation before relying on a specific hook or setting in a cloud session.

## 2. What applies in adapted form: chat apps

### 2.1 Context

In the claude.ai chat app, project instructions and uploaded files play the role of context files (unverified; check the claude.ai help pages). Write project instructions the way you would write a short [context file]({{ '/sections/04-context-files/' | relative_url }}): the rules you keep repeating, not general advice.

#### Worked example: project instructions

```text
You are helping me write internal status notes.
- Lead with the answer, then the blockers.
- No em dashes, no bold lead-ins, no three-part lists.
- End every reply with a "Notes" block: decisions made, open questions, next step.
- If my request is ambiguous, ask one question before writing.
```

### 2.2 Skills and plugins

The same plugin format installs on claude.ai and in Cowork, where a different set of components loads. A skill written for Claude Code can be useful in a chat app as a procedure the model reads. The difference is that nothing checks whether the model followed it. See [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

### 2.3 Every rule is an instruction

In a chat app there is no hook, no test and no gate between the model and you. Every rule sits on the softest rung of the [enforcement ladder]({{ '/sections/17-building-a-harness/' | relative_url }}#3-prefer-a-fix-that-a-program-enforces). Expect it to be followed less consistently than a rule a hook enforces, and plan to check the result yourself.

## 3. What does not apply

Hooks, scripts and local guards need a local client. In a chat app:

| Mechanism | Why it does not apply | What to do instead |
|---|---|---|
| Hooks | No client runs code at points in the model's loop | Put the check in a written checklist and run it yourself |
| Local scripts and linters | Nothing runs on your machine | Run them yourself on the output |
| Permission rules | The model cannot act on your machine | Not needed for the same reason |
| Tests and merge gates | Nothing lands without you | Review the output before you use it |

## 4. Habits carry over everywhere

The habits behind a harness work in any client:

1. Write durable notes, so the next session starts from them and not from memory.
2. Keep the original request, word for word, and check the result against it.
3. Check results yourself before you rely on them.
4. Decide in advance what needs your approval.

The [chat habits page]({{ '/sections/19-chat-and-desktop-clients/harness-habits-in-a-chat-app.html' | relative_url }}) shows each one in practice, with a worked example of the notes habit.

## Examples from my own harness

My harness runs in the terminal clients, so I have no chat-app setup of my own to show. The habits in section 4 come from the terminal harness: durable notes, the original ask kept verbatim, and checks before claims are all rules there, and they work the same way when I use a chat app.

To start: copy the project instructions example above into a chat project and use it for a week.

## Pages in this section

- [Harness habits in a chat app]({{ '/sections/19-chat-and-desktop-clients/harness-habits-in-a-chat-app.html' | relative_url }})

## Sources

- Surfaces and what they share: <https://code.claude.com/docs/en/platforms.md>, verified 2026-11-08.
- Plugins on claude.ai and cloud sessions: <https://code.claude.com/docs/en/plugins/overview.md>, verified 2026-11-08.

{% include page_nav.html %}
