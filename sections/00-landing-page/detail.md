---
title: "Detail"
parent: "Introduction"
nav_order: 1
redirect_from:
  - /sections/00-introduction/detail.html
---

{% include page_nav.html %}

# Introduction in detail

As of 2026-11-09.

The same content serves a 45-minute talk and this set of linked pages. Each topic has a section page with the points said on the talk path, a detail page with the purpose, key points, examples, pitfalls and sources, and, where there is real content, an advanced page with optional deeper material. The [talk slides]({{ '/talk/' | relative_url }}) page lists the slides in order.

Examples from my own harness are described conceptually: what a subsystem does, how it fits, and what you could build. A source marked "verified" with a date was fetched that day and supports the claim. A claim marked "unverified" or "not yet verified against the primary source" has not yet been checked against its primary source.

## The story

The talk runs from the basics to building your own harness.

1. What makes a model agentic, and what a harness is.
2. A quick tour of the building blocks that Claude Code and Codex provide: context files, skills and plugins, hooks and checks, and templates.
3. How the blocks combine into a harness: delegation, coordination and orchestration, context management, verification, human gates, and measurement.
4. How to build your own harness with systems thinking and reflexivity, and how to keep it current as models change.

The basics move fast on the talk path, with their detail on linked pages. Most of the time goes to how the blocks combine, because that is where a harness differs from a default setup.

Two ideas run through the talk.

Systems thinking means treating the model, its instructions, its tools, its checks and the person directing it as one system. When an output is wrong, ask which part of the system let it through, and fix that part.

Reflexivity is my word for a harness that improves itself from its own friction. When an agent makes a mistake that may happen again, or does something that should keep happening, the harness is updated so the next session starts with the change. The change goes in whichever part of the harness fits: a context file, a skill, a template, a hook, a script, a test or a tool. Agents do much of that work themselves. Over time the harness grows into a semi-autonomous system that works the way its owner works.

In practice the two ideas lead to one working principle: enforce as much as possible in code, and rely on context-file instructions only where code cannot do the job. An agent follows an instruction inconsistently over time. It slips more as the context window fills, when it juggles several tasks, and when it switches to something else and forgets to come back. A hook, script, linter, guard, test or CI gate runs the same way every time. [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) orders the options from soft to hard enforcement, and [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}) shows how to apply them. The same page covers the opposite risk: a harness that keeps adding mechanisms until nobody can follow it.

The talk path and the linked pages are the same content at two depths. During the talk I can open any detail page when a question or the room calls for it, and listeners can read the pages afterwards.

## Talk path

| Stop | Minutes | Pages | What it covers |
|---|---:|---|---|
| 1 | 2 | [Introduction]({{ '/' | relative_url }}) | Who the talk is for, the conductor model, how to use the linked pages |
| 2 | 2 | [Agentic AI fundamentals]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}) | The agent loop, tools, and the context window |
| 3 | 3 | [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}) | What a harness is, systems thinking, reflexivity |
| 4 | 6 | [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}), [Context files]({{ '/sections/04-context-files/' | relative_url }}), [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}), [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }}) | The building blocks, one line each with its main use |
| 5 | 4 | [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) | Enforcing rules in code, and the soft-to-hard ladder |
| 6 | 4 | [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}) | Delegation and sub-agents |
| 7 | 4 | [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}), [Orchestration and workflows]({{ '/sections/10-orchestration/' | relative_url }}) | Coordinating sessions, and the two kinds of orchestration |
| 8 | 2 | [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}) | Context window management and memory |
| 9 | 4 | [Testing agent-written code]({{ '/sections/12-testing/' | relative_url }}), [Code review]({{ '/sections/13-code-review/' | relative_url }}) | Testing and code review |
| 10 | 3 | [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}) | Human gates and staying aligned |
| 11 | 2 | [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}) | Logging and evaluation |
| 12 | 5 | [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}) | Building your own harness, and complexity creep |
| 13 | 1 | [Trends and model changes]({{ '/sections/18-trends/' | relative_url }}) | Model changes and trends |
| | 42 | | Talk total |
| | 3 | | Questions |
| | 45 | | Session total |

[Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}) and [Chat, desktop and web clients]({{ '/sections/19-chat-and-desktop-clients/' | relative_url }}) have detail pages only. Stop 9 mentions screenshots in one line, and stop 1 mentions chat and desktop users in one line. Demos come later, after the talk and these pages are settled.

## Purpose

Say who the material is for, what it covers, and how to read it.

## Key points

- The audience ranges from engineers running coding agents to technical and non-technical workers who use desktop or web chat interfaces.
   - To leverage all the tools at one's disposal for AI harness engineering, it's recommended to use Claude Code / Code or other agentic AI CLI clients, though some of what is discussed will apply to desktop and web interfaces to varying degrees.
- The methods apply to personal projects and goals as well as to professional work. As a person's skill and harness grow, the amount of work they can direct grows with them.
  - The main tools are Claude Code, then optionally Codex.
- In a more advanced setup, agents coordinate among themselves on long tasks. The person gives the starting direction and steps in on key decisions, blockers and feedback.
- The aim is to show what is possible and how to start, so each reader can build a harness that fits how they work.

## Pitfalls and lessons

- A copied harness brings its author's habits and problems with it. Build from the problems you hit yourself.

## Sources

The meeting description I wrote for this talk.

## Licence and credits

Prose on this site is licensed under CC BY 4.0, and code under Apache-2.0. The [repository README](https://github.com/sergeiwallace/ai-harness-engineering#licences) maps each licence to its files.

The coordination examples use firstmate, an open-source project by GitHub user kunchenguid (<https://github.com/kunchenguid/firstmate>). I use my own public fork with customizations for my setup (<https://github.com/sergeiwallace/firstmate>).

{% include page_nav.html %}
