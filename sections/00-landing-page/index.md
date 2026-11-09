---
title: "Introduction"
nav_order: 1
has_toc: false
permalink: /
redirect_from:
  - /sections/00-landing-page/
  - /sections/00-introduction/
  - /sections/00-landing-page/detail.html
  - /sections/00-introduction/detail.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Agentic AI Workflows & Harness Engineering

Fundamentals & How to Build Your Own
{: .fs-6 .fw-300 }

![Illustration: a huge many-eyed monster labelled LLM, held by a harness of straps and reins, with a small agent at its head spraying a stream of code labelled your code]({{ '/assets/images/harness-llm-agent-code.jpg' | relative_url }}){: style="width: 100%" }

This wiki is a guide to working with AI agents such as Claude Code and Codex, and to building a harness around them: the context, tools, checks and helpers that make an agent work the way you want to work.

- The same content serves a 45-minute talk and this set of linked pages.
- This page is stop 1 on the talk path, about 2 minutes.

## Who this is for

Anyone who uses AI models for work. The examples come from software engineering, where agents run in a terminal and change code, and most of the ideas also apply, in adapted form, to the chat and desktop apps.

| If you use | Start with | Then read |
|---|---|---|
| Claude Code or Codex in a terminal | [Agentic AI fundamentals]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}) | the building blocks, sections 3 to 7 |
| A chat, desktop or web app | [Desktop, Web & Mobile Apps]({{ '/sections/19-chat-and-desktop-clients/' | relative_url }}) | context files and skills, which have chat-app equivalents |
| Several agents already | [Delegation]({{ '/sections/08-delegation/' | relative_url }}) and [orchestration]({{ '/sections/10-orchestration/' | relative_url }}) | testing, review and human gates |

You do not need a harness to start. The site shows what one is, how the parts fit, and a first week of steps to [build your own]({{ '/sections/17-building-a-harness/' | relative_url }}).

### Key points

- The audience ranges from engineers running coding agents to technical and non-technical workers who use desktop or web chat interfaces.
   - To leverage all the tools at one's disposal for AI harness engineering, it's recommended to use Claude Code / Code or other agentic AI CLI clients, though some of what is discussed will apply to desktop and web interfaces to varying degrees.
- The methods apply to personal projects and goals as well as to professional work. As a person's skill and harness grow, the amount of work they can direct grows with them.
  - The main tools are Claude Code, then optionally Codex.
- In a more advanced setup, agents coordinate among themselves on long tasks. The person gives the starting direction and steps in on key decisions, blockers and feedback.
- The aim is to show what is possible and how to start, so each reader can build a harness that fits how they work.

## What a harness is

A coding agent is a model in a loop: it reads its context, asks for an action, and reads the result ([Agentic AI fundamentals]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }})). The harness is everything you put around that loop so the agent works the way you need it to.

| Part | What it does | Example | Covered in |
|---|---|---|---|
| Context files | tell every session the project's facts and conventions | a `CLAUDE.md` naming the test command | [Context files]({{ '/sections/04-context-files/' | relative_url }}) |
| Skills and templates | give the agent procedures and structures to follow | a release skill that runs the steps in order | [Skills]({{ '/sections/05-skills-and-plugins/' | relative_url }}), [Templates]({{ '/sections/07-templates-and-stubs/' | relative_url }}) |
| Tools | decide what the agent can do | an MCP server for the issue tracker | [Fundamentals: tools]({{ '/sections/01-agentic-ai-fundamentals/tools.html' | relative_url }}) |
| Permissions, hooks and linters | enforce how you want the output to be (conventions, structure, style), and stop the mistakes you have seen, at the moment of each action | a linter that requires a docstring on every public function; a hook that refuses a commit while tests fail | [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) |
| Tests and validation | check the output meets your standards, such as required sections or passing behaviour, and catch what got past the earlier parts, before a change lands | a CI job on every merge | [Testing]({{ '/sections/12-testing/' | relative_url }}) |
| Memory and plans | carry state from one session to the next | a plan file with done, next and watch out | [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}) |
| Coordination | split work across agents and bring it back together | a coordinator that briefs sub-agents | [Delegation]({{ '/sections/08-delegation/' | relative_url }}), [Orchestration]({{ '/sections/10-orchestration/' | relative_url }}) |

The model is the same in every setup; the harness decides what it sees, what it may do, and what gets checked before its work lands. [What is a harness?]({{ '/sections/02-what-is-a-harness/' | relative_url }}) takes the parts in turn.

## The conductor model

![Diagram: the conductor model. You give the goal and direction to a coordinating agent, which sends briefs to a code agent, a tests agent and a review agent; they commit to, run against and read the repository; the coordinator sends decisions and blockers back to you; you adjust the harness after each friction, and the harness is loaded into every agent's session]({{ '/assets/images/00-introduction/conductor-model.svg' | relative_url }}){: style="width: 100%" }
*Figure: the conductor model. Solid arrows are work; dashed arrows are what comes back to you and what you feed into the harness.*

Think of yourself as a conductor with an orchestra. On a long task the parts work together like this:

1. You give a coordinating agent the goal and the direction.
2. The coordinator splits the work and sends each agent a brief.
   - One agent may write the change, another the tests, another a review; the split depends on the task.
3. Each agent works in its own session, and the harness is loaded into every one of them.
4. The coordinator collects the results and sends decisions and blockers back to you.
5. When something goes wrong in a way that could repeat, you adjust the harness, so the next session starts with the fix.

In a simpler setup you are the coordinator yourself and run one agent at a time; the same harness applies.

My own harness appears throughout as one example of what is possible:

- it is described conceptually: what each mechanism does and how it is wired, without the code itself
- a copied harness brings its author's habits and problems with it, so take the ideas and build from the problems you hit yourself.

## The story

The talk runs from the basics to building your own harness.

1. What makes a model agentic, and what a harness is ([fundamentals]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}), [harnesses]({{ '/sections/02-what-is-a-harness/' | relative_url }})).
2. A quick tour of the building blocks that Claude Code and Codex provide: [the clients]({{ '/sections/03-claude-code-and-codex/' | relative_url }}), [context files]({{ '/sections/04-context-files/' | relative_url }}), [skills and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}), [hooks and checks]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) and [templates]({{ '/sections/07-templates-and-stubs/' | relative_url }}).
3. How the blocks combine into a harness: [delegation]({{ '/sections/08-delegation/' | relative_url }}), [coordination]({{ '/sections/09-session-coordination/' | relative_url }}), [orchestration]({{ '/sections/10-orchestration/' | relative_url }}), [context management]({{ '/sections/11-context-window/' | relative_url }}), [testing]({{ '/sections/12-testing/' | relative_url }}), [review]({{ '/sections/13-code-review/' | relative_url }}), [human gates]({{ '/sections/14-human-gates/' | relative_url }}) and [measurement]({{ '/sections/16-logging-and-evaluation/' | relative_url }}).
4. How to [build your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}) with a systems approach, and how to keep it current [as models change]({{ '/sections/18-trends/' | relative_url }}).

The basics move fast on the talk path. Most of the time goes to how the blocks combine, because that is where a harness differs from a default setup.

## A systems approach

The talk takes a systems approach to harness engineering and to working with agentic AI:

- the model, its instructions, its tools, its checks and the person directing it form one system
- when the output is wrong or a gap shows up, the fix goes into the system
- the same approach is applied again to each new issue, often by the agents themselves, so the harness improves with use and the output gets better in a way the system enforces.

Context files and instructions guide the model, but the model applies them unevenly. It drops rules more often:

- when it handles several things at once
- when it switches tasks
- when its context window is nearly full.

Claude Code's own documentation says it treats `CLAUDE.md` "as context, not enforced configuration". So:

- adding another detailed rule to a context file each time the model does something wrong is an anti-pattern
- how you want the output to be, and the fix for each issue, belong in a layer that enforces them.

![Diagram: the systems approach as a loop. A work session hits an issue or gap; you or the agent ask which layer can enforce the fix; the fix goes into layer 2, workflows and tools, layer 3, hooks, scripts and linters, or layer 4, validation and tests, with a context-file line as layer 1, the second line of defence; the next session runs inside the improved system]({{ '/assets/images/00-introduction/systems-approach-loop.svg' | relative_url }}){: style="width: 100%" }
*Figure: each issue goes back into the system. The enforcing layers carry the fix; the instruction layer repeats it for the model.*

A harness enforces its rules in four layers:

1. Instructions in context files, skills and prompts describe the expected behaviour.
   - They shape what the model does by default.
   - Over many sessions they are never enough on their own.
2. Workflows and tools build the steps into what the agent runs.
   - A skill runs the steps in a fixed order; a tool accepts only valid input; a template gives the structure to fill in.
3. Hooks, scripts and linters act at the moment the model tries something.
   - Proactively, they enforce how you want the output to be: naming, structure, style, required sections.
   - Reactively, they refuse or flag a mistake you have seen before.
4. Validation and tests run before a change lands.
   - They check the output meets your standards, and catch what got past the earlier layers.

Layers 2 to 4 do the enforcing. Layer 1 stays as a second line of defence, so the model's default behaviour already matches what the checks require.

- [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers layer 3 in depth.
- [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}) covers the opposite risk: a harness that keeps adding mechanisms until nobody can follow it.

### Worked example: commits with a failing test suite

1. The issue: agents sometimes commit while tests fail, usually late in a long session.
2. The ad hoc fix would be a new context-file line, "run the tests before committing". It holds most days and slips on the long ones.
3. The systems fix goes into the enforcing layers:
   - layer 3: a pre-commit hook runs the suite and refuses the commit on a failure, with a message naming the failing tests
   - layer 4: the CI gate runs the same suite before a merge, for anything committed outside the hook.
4. The context-file line stays as layer 1, and the hook's refusal message repeats it at the moment it applies.
5. Every later session, by any agent, runs inside the same checks.

## How the site is organised

Each section has an overview and, where a subject needs more depth, topic pages under it.

| Page kind | What it holds | Example |
|---|---|---|
| Overview | every idea in the section, each with a worked example; readable on its own | [Agentic AI fundamentals]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}) |
| Topic page | one concept, tool or technique in depth: diagrams, snippets, fuller examples, pitfalls, sources | [The agent loop]({{ '/sections/01-agentic-ai-fundamentals/the-agent-loop.html' | relative_url }}) |
| Sub-topic page | a part of a topic that needs its own page | [How an agent run ends]({{ '/sections/01-agentic-ai-fundamentals/how-a-run-ends.html' | relative_url }}) |

Some sections have several topic pages, and some have none because the overview covers the subject. Links inside the text lead from a term to the page that covers it in more depth.

How to move around:

- every page has previous and next links at the top and bottom, in reading order
- the left sidebar lists every section, with its topic pages nested under it
- the [talk slides]({{ '/talk/' | relative_url }}) page lists the slides in order.

The talk path and the linked pages are the same content at two depths. During the talk I can open any page when a question calls for it, and listeners can read the pages afterwards.

Conventions used on every page:

- Examples from my own harness are described conceptually: what a mechanism does, how it fits, and what you could build.
- A source marked "verified" with a date was fetched that day and supports the claim.
- A claim marked "not yet verified against the primary source" has not yet been checked there.

## Talk path

| Stop | Minutes | Pages | What it covers |
|---|---:|---|---|
| 1 | 2 | [Introduction]({{ '/' | relative_url }}) | Who the talk is for, the conductor model, how to use the linked pages |
| 2 | 2 | [Agentic AI fundamentals]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}) | The agent loop, tools, and the context window |
| 3 | 3 | [What is a harness?]({{ '/sections/02-what-is-a-harness/' | relative_url }}) | What a harness is, and a systems approach to building one |
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

Two sections are for reading only, off the talk path:

- [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}), which stop 9 mentions in one line about screenshots;
- [Desktop, Web & Mobile Apps]({{ '/sections/19-chat-and-desktop-clients/' | relative_url }}), which stop 1 mentions in one line for chat and desktop users.

Demos come later, after the talk and these pages are settled.

## Pitfalls and lessons

| Pitfall | What happens | Instead |
|---|---|---|
| Copying someone else's harness | it brings its author's habits and problems with it | build from the problems you hit yourself |
| A new context-file rule for every mistake | the file grows, and the model follows each rule less reliably | put the fix in an enforcing layer, and keep one line explaining it |
| Adding mechanisms faster than you remove them | nobody can follow the harness, including its agents | remove what no longer earns its place ([Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }})) |
| Trusting an agent's summary of its own work | reported work that was never done goes unnoticed | check the tool calls, the files and the test results |

## Sources

- The meeting description I wrote for this talk.
- "Context, not enforced configuration": Claude Code's documentation on memory, [Claude Code docs: How Claude remembers your project](https://code.claude.com/docs/en/memory.md), verified 2026-11-09.

## Licence and credits

| Content | Licence |
|---|---|
| Prose and diagrams | CC BY 4.0 |
| Code and configuration snippets | Apache-2.0 |

The [repository README](https://github.com/sergeiwallace/ai-harness-engineering#licences) maps each licence to its files.

The coordination examples use [firstmate](https://github.com/kunchenguid/firstmate), an open-source project by GitHub user kunchenguid. I use [my own public fork of firstmate](https://github.com/sergeiwallace/firstmate) with customizations for my setup.

## Contents

{% include page_toc.html %}

{% include page_nav.html %}
