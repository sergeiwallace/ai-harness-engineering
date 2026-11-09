---
title: "Orchestration frameworks"
parent: "Programmatic workflows"
grand_parent: "Orchestration and workflows"
nav_order: 1
---

{% include page_nav.html %}

# Orchestration frameworks

As of 2026-11-09.

A sub-topic of [programmatic workflows]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}). When the plan moves into code, you can write the loop yourself, use your coding agent's own workflow feature, or use a framework. This page compares the main options by the model they use for a plan, and gives a way to choose.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Three ways to put a plan in code

| Approach | The plan is | Good when | Cost |
|---|---|---|---|
| Your own script | Plain code calling a model or a CLI | A few steps, one model, you own the runtime | You write retries, state and resume |
| Your coding agent's workflow feature | A script the agent writes and runs, such as Claude Code's dynamic workflows | The work is coding work inside that agent | Tied to that agent |
| A framework | A graph, a crew or a set of agents with handoffs | A product feature, many tools, several models, long-lived state | A dependency and its concepts |

> For orchestration that helps you write code, a script or your coding agent's workflow feature is often enough. Frameworks add more when the orchestration ships as part of a product.
{: .note }

## The frameworks

### LangGraph

LangGraph describes a workflow as a graph of steps with shared state, retries and human approval points. Nodes are functions, edges say what runs next, and conditional edges branch on state. A checkpointer saves state after each step, which is what makes resume and human approval possible: the graph pauses, a person approves, and it continues from the saved state. The [programmatic workflows page]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}) has a short example.

![A sequence diagram. The graph asks the model for a draft and gets it back, then saves its state to a checkpoint. In an optional fragment it requests approval from a person, who approves the next day. The graph loads its state from the checkpoint, gets the draft back, and asks the model to send it.]({{ '/assets/images/10-orchestration/graph-approval.svg' | relative_url }}){: style="width:100%"}

*Figure: a checkpointed graph waits for a person without holding a process open. Made with archify.*

### OpenAI Agents SDK

| Concept | What it is |
|---|---|
| Agent | A model with instructions and tools |
| Handoff | One agent passing the conversation to another |
| Guardrail | A check on inputs or outputs that can stop a run |
| Tracing | A record of each run's steps |

The plan is the set of agents and the handoffs allowed between them.

### CrewAI

- A crew of agents, each with a role, such as researcher, writer, editor.
- Tasks assigned to the agents.
- A process that orders the tasks.

It suits work that reads naturally as a team.

### Google Agent Development Kit

- Model-driven agents that decide their own next step.
- Workflow agents that run sub-agents in a fixed pattern:
  - in sequence,
  - in parallel,
  - or in a loop.

A fixed plan and model judgment can be combined in one tree of agents.

### Microsoft Agent Framework

- It replaced the archived AutoGen.
- It covers single agents and multi-agent workflows.
- It fits teams already working in Microsoft's stack.

## Versions

| Framework | PyPI version on 2026-11-08 |
|---|---|
| LangGraph | 1.2.20 |
| OpenAI Agents SDK | 0.24.0 |
| CrewAI | 1.15.38 |
| Google ADK | 2.14.0 |

Not re-checked since 2026-11-08. These projects release often; read the changelog before relying on an example.

## Choosing

1. Is this coding work in your own repository? Use your coding agent's workflow feature or a short script.
2. Does the plan need to pause for a person and resume later? Pick something with saved state: a checkpointed graph, or your own script that writes each step to a file.
3. Is the plan mostly "which agent handles this next"? A handoff model fits.
4. Is it a fixed sequence with a few judgment points? Workflow agents, or a graph with conditional edges.

#### Example: the same job three ways

A nightly job reads new support tickets, classifies each, drafts replies for two categories and sends the rest to a person.

| Built as | What it looks like |
|---|---|
| A script | A loop over tickets: one model call to classify, one to draft, a file per ticket as state |
| A graph | Nodes `classify`, `draft`, `human_review`; a conditional edge on the category; a checkpointer so a person's review can resume the run |
| Agents with handoffs | A triage agent that hands each ticket to a drafting agent or a human-review agent |

The script is the easiest to read. The graph is the easiest to pause for a person and resume. The handoff model is the easiest to extend with a new kind of agent.

## Sources

- LangGraph: <https://docs.langchain.com/oss/python/langgraph/overview>, verified 2026-11-08.
- AutoGen archived: <https://github.com/microsoft/autogen>, verified 2026-11-08.
- OpenAI Agents SDK: <https://openai.github.io/openai-agents-python/>. CrewAI: <https://docs.crewai.com/>. Google ADK: <https://google.github.io/adk-docs/>. Not re-verified on 2026-11-09.

{% include page_nav.html %}
