---
title: "Programmatic workflows"
parent: "Orchestration and workflows"
nav_order: 2
has_children: true
has_toc: false
redirect_from:
  - /sections/10-orchestration/advanced.html
---

{% include page_nav.html %}

# Programmatic workflows

As of 2026-11-09.

Programmatic orchestration holds the plan in code. A script or a graph decides which agent runs when, with what input, and what happens after. This page covers Claude Code's dynamic workflows, graph frameworks such as LangGraph, the other frameworks in use, and how to design a workflow that resumes cleanly. The [section overview]({{ '/sections/10-orchestration/' | relative_url }}) compares it with [prompt-driven orchestration]({{ '/sections/10-orchestration/prompt-driven-orchestration.html' | relative_url }}). A sub-topic page compares [orchestration frameworks]({{ '/sections/10-orchestration/orchestration-frameworks.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Why put the plan in code

| Property | Plan in a prompt | Plan in code |
|---|---|---|
| Context cost | The plan and its bookkeeping sit in the lead's context | None: the script holds them |
| Scale | A handful of agents | Hundreds |
| Repeatability | Varies run to run | Same steps every run |
| Resume after failure | Rebuild from notes | Restart from the last finished step |
| Judgment between steps | Full | Only where the script calls a model to decide |

It fits work with known steps, many parallel agents, or a need to resume after a failure.

## Claude Code dynamic workflows

Claude Code's dynamic workflows are JavaScript scripts that run many sub-agents.

| Property | Value |
|---|---|
| Language | JavaScript |
| Plan held in | The script; it does not take up Claude's context |
| Agents at once (default) | 16 |
| Agents per run (default) | 1,000 |
| Watching a run | `/workflows` |

### Starting one

- Ask Claude for a workflow in plain words.
- Or use the `ultracode` keyword, which asks Claude to write one.
- Or run the bundled `/deep-research` workflow.

### The shape of a script

The sketch below shows the shape, not the exact API; the [workflows docs](https://code.claude.com/docs/en/workflows.md) have the real function names.

```javascript
// Shape of a fan-out workflow: list, run in batches, join, report.
const files = await listFiles("src/**/*.py");
const batches = chunk(files, 10);

const verdicts = await Promise.all(
  batches.map((batch) =>
    runAgent({
      prompt: `Check these files against the logging rule. Return JSON
               {file, ok, reason} per file.\n${batch.join("\n")}`,
    })
  )
);

await writeReport(verdicts.flat().filter((v) => !v.ok));
```

### Watching and controlling a run

`/workflows` watches, pauses and saves dynamic workflows. A paused run can be inspected before it spends more.

> A workflow can start hundreds of agents. Check the default limits, and think about the cost of a full run, before starting one on a large input.
{: .warning }

## Graph frameworks

### LangGraph

LangGraph describes a workflow as a graph of steps with shared state, retries and human approval points. Each node is a function, often one that calls a model; edges say what runs next, and conditional edges pick a branch from the state.

A minimal LangGraph-style graph in Python:

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    task: str
    result: str
    tries: int

def work(state: State) -> dict:
    return {"result": call_model(state["task"]), "tries": state["tries"] + 1}

def route(state: State) -> str:
    if passes_checks(state["result"]):
        return END
    return "work" if state["tries"] < 3 else "human_review"

graph = StateGraph(State)
graph.add_node("work", work)
graph.add_node("human_review", human_review)
graph.add_edge(START, "work")
graph.add_conditional_edges("work", route)
app = graph.compile()
```

The exit condition (`passes_checks`) and the limit (`tries < 3`) are both in code, where nobody can forget them.

### Other frameworks

[Orchestration frameworks]({{ '/sections/10-orchestration/orchestration-frameworks.html' | relative_url }}) compares these in more depth.

| Framework | Notes | Version on PyPI, 2026-11-08 |
|---|---|---|
| LangGraph | Graph of steps, shared state, checkpoints | 1.2.20 |
| OpenAI Agents SDK | Agents, handoffs between them, guardrails | 0.24.0 |
| CrewAI | Role-based crews of agents | 1.15.38 |
| Google Agent Development Kit | Agents and workflow agents | 2.14.0 |
| Microsoft Agent Framework | Replaced the archived AutoGen | |

Versions were not re-checked since 2026-11-08.

## Designing for resume

![A sequence diagram. The script starts agent batches; the agents write checkpoints 1 to 13, then batch 14 fails. In an optional fragment the script lists the done checkpoint files, gets 1 to 13 back, and restarts at 14; the agents write 14 to 20. The script joins all 20 into the report.]({{ '/assets/images/10-orchestration/workflow-resume.svg' | relative_url }}){: style="width:100%"}

*Figure: a workflow script resumes from its checkpoints after a failure. Made with archify.*

#### Make each step idempotent

Running a step twice gives the same result as running it once. A step that writes "batch 14 done" and then its output can be skipped on resume; a step that appends to a shared file without a marker duplicates on resume.

#### Persist state between steps

Write each step's output to a file or a checkpoint store before the next step starts. LangGraph's checkpointers do this for graphs.

#### Keep judgment in named places

Where a step needs a model to decide, make it a node with a clear input and a constrained output, such as `pass` or `fail` with a reason, so the script can branch on it.

<details markdown="1">
<summary>Worked example: resuming a 200-file review</summary>

1. The run checks 200 files in 20 batches.
   - Each finished batch writes `out/batch-NN.json`.
2. Batch 14's agent hits a rate limit and the run stops.
3. The script is rerun.
   - It lists which batch files exist.
   - It skips 1 to 13 and starts at 14.
4. The report step reads all 20 files at the end.

The figure above shows the same run.

</details>

## Examples from my own harness

- Most of my harness's orchestration is prompt-driven, through coordinator agents and skills.
- The pieces that became code are the ones that ran the same way many times.
  - A pipeline whose state lives in a validated state file, so any agent that reads it can resume a stage.

To start: take one chain you have run by hand three times the same way, and write it as a script that saves each step's output to a file.

## Sources

- Dynamic workflows, limits, `/workflows`, `ultracode`, `/deep-research`: <https://code.claude.com/docs/en/workflows.md>, verified 2026-11-08.
- LangGraph: <https://docs.langchain.com/oss/python/langgraph/overview>, verified 2026-11-08.
- AutoGen archived: <https://github.com/microsoft/autogen>, verified 2026-11-08.

{% include page_nav.html %}
