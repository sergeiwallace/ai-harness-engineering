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

The Claude Code desktop app and the web version run "the same underlying engine" as the terminal client. Shared across the local surfaces:

- Configuration, including settings files.
- Project memory, such as `CLAUDE.md`.
- MCP servers.

So a `CLAUDE.md`, a skill or an MCP server you set up for the terminal is there in the desktop app too, and the enforcing layers of the harness carry over with it.

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

In the claude.ai chat app, project instructions and uploaded files play the role of context files (unverified; check the claude.ai help pages). Keep project instructions as short as a good [context file]({{ '/sections/04-context-files/' | relative_url }}): the few rules that apply to every chat in the project, such as the audience, the format and how to end a reply. A long list is followed less consistently here too, and a chat app has no hook to catch what the model misses, so resist adding a new line after every bad answer.

#### Worked example: project instructions

```text
You are helping me write internal status notes.
- Lead with the answer, then the blockers.
- No em dashes, no bold lead-ins, no three-part lists.
- End every reply with a "Notes" block: decisions made, open questions, next step.
- If my request is ambiguous, ask one question before writing.
```

### 2.2 Skills and plugins

The same plugin format installs on claude.ai and in Cowork, where a different set of components loads.

| Part of a skill or plugin | In Claude Code | In a chat app |
|---|---|---|
| The procedure in `SKILL.md` | The agent follows it, and a hook can check the result | The model reads it as guidance |
| Scripts the skill runs | Run on your machine | Do not run on your machine |
| Hooks the plugin adds | Run on matching events | Do not apply |

See [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

### 2.3 Every rule is an instruction

In a chat app there is no hook, no test and no gate between the model and you, so every rule sits in layer 1 of the [four layers]({{ '/sections/17-building-a-harness/' | relative_url }}#11-the-four-layers).

- Expect each rule to be followed less consistently than a rule a hook enforces.
  - Most of all in a long conversation, or one that switches between tasks.
- You take over the enforcing layers.
  - A written checklist you run on each answer stands in for hooks and tests.
  - Your own approval stands in for gates.

### 2.4 A systems approach without enforcement

The four layers still guide how you set up a chat project. You cannot add hooks, but you can shape the workflow so the model has less to juggle, and you can make your own checks routine.

| Layer | In a coding agent | In a chat app |
|---|---|---|
| 1. Instructions | Context files, skill text | Project instructions, kept short |
| 2. Workflows and tools | Skills that run fixed steps, templates | A saved prompt template per kind of task; one chat per task, so the model does not switch between tasks; a new chat when a conversation grows long |
| 3. Hooks, scripts and linters | Checks at the moment of action | None; ask for output in a fixed structure that is quick for you to scan |
| 4. Validation and tests | Tests, gates, review | Your checklist on each answer, or a fresh chat that reviews the answer against the checklist |

<!-- IMAGE NEEDED: the four layers side by side for a coding agent and a chat app, showing layers 3 and 4 handed over to the person in the chat app. -->

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
5. Keep one task per session, and start a new one when a conversation grows long.

The [chat habits page]({{ '/sections/19-chat-and-desktop-clients/harness-habits-in-a-chat-app.html' | relative_url }}) shows each one in practice, with a worked example of the notes habit.

## Examples from my own harness

- My harness runs in the terminal clients, so I have no chat-app setup of my own to show.
- The habits in section 4 come from the terminal harness.
  - Durable notes, the original ask kept verbatim, and checks before claims are each enforced there by a workflow, a template or a check.
  - In a chat app I carry them out myself.

To start: copy the project instructions example above into a chat project and use it for a week.

## Pages in this section

- [Harness habits in a chat app]({{ '/sections/19-chat-and-desktop-clients/harness-habits-in-a-chat-app.html' | relative_url }})

## Sources

- Surfaces and what they share: <https://code.claude.com/docs/en/platforms.md>, verified 2026-11-08.
- Plugins on claude.ai and cloud sessions: <https://code.claude.com/docs/en/plugins/overview.md>, verified 2026-11-08.

{% include page_nav.html %}
