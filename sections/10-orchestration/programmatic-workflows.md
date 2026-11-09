---
title: "Programmatic workflows"
parent: "Orchestration and workflows"
nav_order: 2
has_children: true
has_toc: false
redirect_from:
  - /sections/10-orchestration/advanced.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Programmatic workflows

Programmatic orchestration holds the plan in code: a script or a graph decides which agent runs when, with what input, and what happens after:

- This page covers Claude Code's dynamic workflows in depth (starting, the script, limits, resume, saving, cost), graph frameworks such as LangGraph, and how to design a workflow that resumes cleanly.
- The [section overview]({{ '/sections/10-orchestration/' | relative_url }}) compares it with [prompt-driven orchestration]({{ '/sections/10-orchestration/prompt-driven-orchestration.html' | relative_url }}); the sub-topic page compares [orchestration frameworks]({{ '/sections/10-orchestration/orchestration-frameworks.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Why put the plan in code

| Property | Plan in a prompt | Plan in code |
|---|---|---|
| Context cost | The plan and its bookkeeping sit in the lead's context | The script holds them; Claude's context holds only the final answer |
| Scale | A handful of agents | Dozens to hundreds per run |
| Repeatability | Varies run to run | The same orchestration every run, and it can be saved |
| Resume after a failure | Rebuild from notes | Finished agents return saved results |
| Judgment between steps | Full | Only where the script calls an agent to decide |
| Quality patterns | Whatever the lead remembers to do | Built in: for example, independent agents that cross-check each finding before it is reported |

It fits work with known steps, many parallel agents, or a need to resume after a failure.

## Claude Code dynamic workflows

A dynamic workflow is a JavaScript script that orchestrates many sub-agents.

- Claude writes the script for the task you describe.
- A runtime executes it in the background, separate from your conversation, while your session stays responsive.
- Intermediate results live in script variables; Claude's context gets only the final answer.
- Available on all paid plans, with Anthropic API access, and on Amazon Bedrock, Google Cloud's Agent Platform and Microsoft Foundry; on Pro, turn them on in `/config`.

### Starting one

| Route | How |
|---|---|
| Ask in plain words | "use a workflow to audit every route handler under src/routes/ for missing auth checks" |
| The keyword | Include `ultracode` in a prompt you type; `Option+W` / `Alt+W` dismisses it for that prompt |
| For every substantive task | `/effort ultracode`, or start with `claude --effort ultracode`; turn off with `/effort ultracode off` |
| A bundled workflow | `/deep-research <question>`: fans out web searches, cross-checks sources, and returns a cited report |
| A saved workflow | `/<name>`, from `.claude/workflows/` or `~/.claude/workflows/` |

The keyword starts a workflow only from a prompt a person typed; it does nothing in a `claude -p` prompt, a scheduled task or a relayed comment.

### Approving the plan

Before a run, Claude Code shows the planned phases and asks you to approve.

| Permission mode | When you are asked |
|---|---|
| Auto | First launch only; skipped when ultracode is on |
| Manual, accept edits | Every run, unless you chose "don't ask again" for that saved workflow in this project |
| Bypass permissions, `claude -p`, Agent SDK | Not asked; in `-p` and the SDK, an allow rule such as `Workflow` or `Workflow(<name>)` lets it start |

The agents a workflow starts still go through your permission rules, so add the tools they need to your allow rules before a long run.

### The script

A saved workflow has a `meta` block and a body of plain JavaScript with top-level `await`. This is the shape the Claude Code docs show:

```javascript
export const meta = {
  name: 'audit-routes',
  description: 'Audit every route handler for missing auth checks',
}

const found = await agent('List every .ts file under src/routes/.', {
  schema: { type: 'object', required: ['files'],
            properties: { files: { type: 'array', items: { type: 'string' } } } },
})

const audits = await pipeline(found.files, file =>
  agent(`Audit ${file} for missing authentication checks.`, { label: file }),
)

return audits.filter(Boolean)
```

| Function or global | What it does |
|---|---|
| `agent(prompt, options)` | Runs one sub-agent; with a `schema`, returns JSON of that shape; resolves to `null` if stopped or after an unrecoverable error |
| `pipeline(items, fn)` | Runs one agent per item, keeping `null` for any that failed |
| `parallel(tasks)` | Runs a set of agent tasks at once and waits for all |
| `phase(title)` | Groups the agents that follow under a title in the progress view |
| `log(message)` | Shows a message above the phases |
| `args` | Input passed when the workflow is invoked, such as a list of issue numbers |

Rules the runtime applies to the script:

- No module loading: a script containing `import()` fails before it starts.
- No direct file or shell access from the script; agents do that work.
- `Date.now()`, `Math.random()` and a no-argument `new Date()` throw, so a relaunched run repeats the same calls. Pass a timestamp in through `args`.

### Limits

| Limit | Default | Change it with |
|---|---|---|
| Agents running at once | 16, fewer on machines with fewer CPUs | `CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS` (1 to 256) |
| Agents per run | 1,000 | Fixed |
| Items in one `parallel()` or `pipeline()` call | 4,096 | Fixed: a longer list is rejected with an error |
| Mid-run input from you | None | Run each stage as its own workflow if you want to sign off between stages |
| Structured-output retries | 5 | `MAX_STRUCTURED_OUTPUT_RETRIES` |
| Size guideline Claude aims for | `medium` (under 10 agents); `small` on Pro | `/config workflowSizeGuideline=small` (or `large`, `unrestricted`) |

### Watching and controlling a run

`/workflows` lists runs; select one to open its progress view.

<!-- IMAGE NEEDED: the Claude Code /workflows progress view with phases and agent counts; search: claude code workflows progress view, claude code dynamic workflows screenshot, ultracode workflow -->

| Key | Action |
|---|---|
| `Enter` / `→` | Drill into a phase, then an agent |
| `p` | Pause or resume the run |
| `x` | Stop the selected agent, or the whole run |
| `r` | Restart a running agent |
| `s` | Save the run's script as a command |
| `f` | Filter agents by status |

> A workflow can start hundreds of agents, and a run counts toward your plan's usage. Claude Code shows a `Large workflow` warning above 25 agents or 1.5 million projected tokens. Try a large task on a small slice first.
{: .warning }

### Resume

Resume works within the same session:

- A paused run resumes with `p` in `/workflows`; for a stopped run, ask Claude to relaunch it.
- The runtime replays the run in the order agents start:

| Agent's state in the last run | On relaunch |
|---|---|
| Completed, same prompt | Returns its saved result |
| Completed, but an earlier agent's prompt or result changed | Runs again, and so does every agent after the first difference |
| Running when you stopped | Starts over |
| Failed, or stopped on its own with `x` | Runs again, and so does every agent started after it |

If you leave the session without moving it to the background, the run stops; `claude --resume` and a relaunch pick it up from saved results. An agent whose output stops arriving restarts automatically, up to five times.

### Saving and sharing

1. In `/workflows`, select the run and press `s`.
2. Choose `.claude/workflows/` in the project (shared with the repository) or `~/.claude/workflows/` (only you).
3. Run it later as `/<name>`; a plugin can ship workflows in a `workflows/` directory, namespaced as `/<plugin>:<name>`.

To turn workflows off: the `/config` toggle, `"disableWorkflows": true` in settings, or `CLAUDE_CODE_DISABLE_WORKFLOWS=1`.

## Graph frameworks

### LangGraph

LangGraph describes a workflow as a graph of steps with shared state, retries and human approval points.

- Nodes are functions, often ones that call a model.
- Edges say what runs next; conditional edges branch on the state.
- A checkpointer saves state after each step, which makes resume and human approval possible.

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

The exit condition (`passes_checks`) and the limit (`tries < 3`) are both in code, where nobody can forget them. `call_model`, `passes_checks` and `human_review` stand for your own functions.

### Other frameworks

[Orchestration frameworks]({{ '/sections/10-orchestration/orchestration-frameworks.html' | relative_url }}) compares these in more depth.

| Framework | Model of a plan | Version on PyPI, 2026-11-08 |
|---|---|---|
| LangGraph | A graph of steps with shared state and checkpoints | 1.2.20 |
| OpenAI Agents SDK | Agents and the handoffs allowed between them | 0.24.0 |
| CrewAI | A crew of role-based agents and a task process | 1.15.38 |
| Google Agent Development Kit | Model-driven agents plus sequential, parallel and loop workflow agents | 2.14.0 |
| Microsoft Agent Framework | Agents and multi-agent workflows; replaced the archived AutoGen | |

Versions were not re-checked since 2026-11-08.

## Designing for resume

![A sequence diagram. The script starts agent batches; the agents write checkpoints 1 to 13, then batch 14 fails. In an optional fragment the script lists the done checkpoint files, gets 1 to 13 back, and restarts at 14; the agents write 14 to 20. The script joins all 20 into the report.]({{ '/assets/images/10-orchestration/workflow-resume.svg' | relative_url }}){: style="width:100%"}

*Figure: a script resumes from its checkpoints after a failure. Made with archify.*

Claude Code's runtime tracks each agent's result for you. A script you write yourself, or a graph framework without a checkpointer, needs the same three properties built in.

#### Make each step idempotent

Running a step twice gives the same result as running it once:

- A step that writes "batch 14 done" with its output can be skipped on resume; a step that appends to a shared file without a marker duplicates on resume.
- An agent that stalls and restarts leaves the files it already changed, so the same rule applies inside a dynamic workflow.

#### Persist state between steps

Write each step's output to a file or a checkpoint store before the next step starts. LangGraph's checkpointers do this for graphs.

#### Keep judgment in named places

Where a step needs a model to decide, make it a node or an `agent()` call with a clear input and a constrained output (a `schema` such as `pass` or `fail` with a reason), so the script can branch on it.

<details markdown="1">
<summary>Worked example: resuming a 200-file review in your own script</summary>

1. The run checks 200 files in 20 batches.
   - Each finished batch writes `out/batch-NN.json`.
2. Batch 14's agent hits a rate limit and the run stops.
3. The script is rerun.
   - It lists which batch files exist.
   - It skips 1 to 13 and starts at 14.
4. The report step reads all 20 files at the end.

</details>

## Examples from my own harness

- Most of my harness's orchestration is prompt-driven, through coordinator agents and skills.
- The pieces that became code are the ones that ran the same way many times.
  - A pipeline whose state lives in a validated state file, so any agent that reads it can resume a stage.

To start: ask Claude Code to "use a workflow" for one task you would otherwise run by hand over many files, watch it in `/workflows`, and save it with `s` if it did what you wanted.

## Sources

- Dynamic workflows: script API, limits, approval, resume, saving, cost and size guideline: [Orchestrate subagents at scale with dynamic workflows (Claude Code docs)](https://code.claude.com/docs/en/workflows.md), verified 2026-11-09.
- LangGraph: [LangGraph overview (LangChain docs)](https://docs.langchain.com/oss/python/langgraph/overview), verified 2026-11-08.
- AutoGen archived: [microsoft/autogen on GitHub](https://github.com/microsoft/autogen), verified 2026-11-08.

{% include page_nav.html %}
