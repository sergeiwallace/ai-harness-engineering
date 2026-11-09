---
title: "Orchestration frameworks"
parent: "Programmatic workflows"
grand_parent: "Orchestration and workflows"
nav_order: 1
---

{% include page_nav.html %}

# Orchestration frameworks

As of 2026-11-09.

A sub-topic of [programmatic workflows]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}). When the plan moves into code, you can write the loop yourself, use your coding agent's own workflow feature, or use a framework. This page describes the main frameworks by how each one models a plan, compares them, and gives a way to choose.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Three ways to put a plan in code

| Approach | The plan is | Good when | Cost |
|---|---|---|---|
| Your own script | Plain code calling a model API or a CLI | A few steps, one model, you own the runtime | You write retries, state and resume |
| Your coding agent's workflow feature | A script the agent writes and runs, such as Claude Code's dynamic workflows | The work is coding work inside that agent | Tied to that agent |
| A framework | A graph, a crew or a set of agents with handoffs | A product feature, many tools, several models, long-lived state | A dependency and its concepts |

> For orchestration that helps you write code, a script or your coding agent's workflow feature is often enough. Frameworks add more when the orchestration ships as part of a product.
{: .note }

## The frameworks

### LangGraph

LangGraph describes a workflow as a graph of steps with shared state, retries and human approval points.

| Concept | What it is |
|---|---|
| State | A typed object every node reads and updates |
| Node | A function, often one that calls a model |
| Edge, conditional edge | What runs next, fixed or chosen from the state |
| Checkpointer | Saves state after each step, so a run can pause for a person and resume |

The [programmatic workflows page]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}#langgraph) has a short example.

![A sequence diagram. The graph asks the model for a draft and gets it back, then saves its state to a checkpoint. In an optional fragment it requests approval from a person, who approves the next day. The graph loads its state from the checkpoint, gets the draft back, and asks the model to send it.]({{ '/assets/images/10-orchestration/graph-approval.svg' | relative_url }}){: style="width:100%"}

*Figure: a checkpointed graph waits for a person without holding a process open. Made with archify.*

### OpenAI Agents SDK

The SDK describes itself as having "a very small set of primitives".

| Concept | What it is |
|---|---|
| Agents | "LLMs equipped with instructions and tools" |
| Agents as tools, handoffs | One agent delegating to another for a specific task |
| Guardrails | Validation of agent inputs and outputs |
| Sessions | A memory layer that keeps working context within an agent loop |
| Tracing | A built-in record of each run, for debugging and monitoring |
| Human in the loop | Built-in ways to involve a person during a run |

The plan is the set of agents and the handoffs allowed between them.

### CrewAI

| Concept | What it is |
|---|---|
| Agents | Role-based agents with tools, memory, knowledge and typed outputs |
| Tasks | Units of work assigned to agents, with guardrails, callbacks and human review steps |
| Crews and processes | A team of agents, ordered by a sequential, hierarchical or hybrid process |
| Flows | Start, listen and router steps with managed state, which can persist and resume long-running runs |

It suits work that reads naturally as a team: a researcher, a writer, an editor.

### Google Agent Development Kit

- Simple agents, where the model decides its next step.
- Template workflows that run sub-agents in a fixed pattern:
  - sequential,
  - parallel,
  - loop,
  - or a custom template.
- Available in Python, TypeScript, Go, Java and Kotlin.

A fixed plan and model judgment can be combined in one tree of agents.

### Microsoft Agent Framework

- Agents and graph-based multi-agent workflows, with "sequential, concurrent, handoff, and group collaboration patterns".
- Workflow features include checkpointing, streaming, human in the loop and time travel.
- Python and .NET, with a separate Go SDK.
- Migration guides from Semantic Kernel and from AutoGen, which is archived.

## Side by side

| | LangGraph | OpenAI Agents SDK | CrewAI | Google ADK | Microsoft Agent Framework |
|---|---|---|---|---|---|
| The plan is | A graph | Agents and handoffs | A crew and a process, or a flow | A tree of agents and workflow templates | A graph of agents |
| Pause for a person | Checkpointer | Human-in-the-loop support | Human review steps | Not checked here | Human in the loop |
| Resume a long run | From a checkpoint | Sessions keep context | Flows persist and resume | Not checked here | Checkpointing |
| Languages | Python, JavaScript | Python | Python | Python, TypeScript, Go, Java, Kotlin | Python, .NET, Go |
| PyPI version, 2026-11-08 | 1.2.20 | 0.24.0 | 1.15.38 | 2.14.0 | |

Versions were not re-checked since 2026-11-08. These projects release often; read the changelog before relying on an example.

## Choosing

1. Is this coding work in your own repository?
   - Use your coding agent's workflow feature or a short script.
2. Does the plan need to pause for a person and resume later?
   - Pick something with saved state: a checkpointed graph, a framework with checkpointing, or your own script that writes each step to a file.
3. Is the plan mostly "which agent handles this next"?
   - A handoff model fits.
4. Is it a fixed sequence with a few judgment points?
   - Workflow templates, or a graph with conditional edges.

#### Example: the same job three ways

A nightly job reads new support tickets, classifies each, drafts replies for two categories and sends the rest to a person.

| Built as | What it looks like |
|---|---|
| A script | A loop over tickets: one model call to classify, one to draft, a file per ticket as state |
| A graph | Nodes `classify`, `draft`, `human_review`; a conditional edge on the category; a checkpointer so a person's review can resume the run |
| Agents with handoffs | A triage agent that hands each ticket to a drafting agent or a human-review agent |

- The script is the easiest to read.
- The graph is the easiest to pause for a person and resume.
- The handoff model is the easiest to extend with a new kind of agent.

## Pitfalls and lessons

| Pitfall | What helps |
|---|---|
| A framework adopted for a three-step script | Start with a script; move when you need saved state or many agent types |
| Examples copied from an older release | Check the version and the changelog first |
| A pause for a person that holds a process open for days | Checkpoint the state and end the process |
| The exit condition left to a model | Put the condition and the limit in code, as in the graph example |

## Sources

- LangGraph: <https://docs.langchain.com/oss/python/langgraph/overview>, verified 2026-11-08.
- OpenAI Agents SDK: <https://openai.github.io/openai-agents-python/>, verified 2026-11-09.
- CrewAI: <https://docs.crewai.com/>, verified 2026-11-09.
- Google Agent Development Kit: <https://adk.dev/> (formerly google.github.io/adk-docs), verified 2026-11-09.
- Microsoft Agent Framework: <https://github.com/microsoft/agent-framework>, verified 2026-11-09. AutoGen archived: <https://github.com/microsoft/autogen>, verified 2026-11-08.

{% include page_nav.html %}
