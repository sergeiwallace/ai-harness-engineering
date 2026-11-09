---
title: "The agent loop"
parent: "Agentic AI fundamentals"
nav_order: 1
has_children: true
has_toc: false
---

{% include page_nav.html %}

# The agent loop

As of 2026-11-09.

The agent loop is the small program that turns a language model into an agent: send the context, read back a request for an action, carry the action out, add the result, repeat. This page takes the loop apart at the level of the messages that pass between a client and a model, shows a minimal loop in code, and marks the points where a harness attaches. The [Agentic AI fundamentals overview]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}) is the short version, and [How an agent run ends]({{ '/sections/01-agentic-ai-fundamentals/how-a-run-ends.html' | relative_url }}) covers stopping.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The loop in one picture

![Diagram: a coding agent as one tall map. You give the task to the client; the client loads context files and memory and sends the whole context to the model; the model's tool call passes a permission gate, which consults hooks; the tool runner edits files, runs the shell or fetches from the web and MCP servers; the tool result returns to the model]({{ '/assets/images/01-agentic-ai-fundamentals/agent-loop-map.svg' | relative_url }}){: style="width: 100%" }
*Figure: the loop as parts, top to bottom. The dashed arrow on the right closes the loop: each tool result goes back into the model's context.*

Read the figure from the top. The model only ever does one thing: it reads the context and writes the next message. Everything else, running tools, checking permissions, firing hooks, trimming output, saving the transcript, is the client's work. The client's half of the loop is software you can configure, and harness engineering is the work of configuring it.

## Messages and content blocks

### The conversation as a list

A model API does not keep a conversation. Each call sends the whole conversation as a list of messages, each with a role. In the Claude Messages API the roles are `user` and `assistant`, and the system instructions travel as a separate `system` field. A message's content is a list of blocks, and three block types carry the loop:

| Block | Written by | What it holds |
|---|---|---|
| `text` | either side | prose: the task, an explanation, a final answer |
| `tool_use` | the model, in an `assistant` message | a request: an id, a tool name and its input |
| `tool_result` | the client, in the next `user` message | the outcome of one request, tagged with the same id |

The id is what ties a result to its request. When the model asks for two tools in one turn, the client returns two results in one message, each tagged with the id of the call it answers.

### One turn, written out

The exchange behind the test-fixing example on the overview, trimmed to its shape:

```json
[
  {"role": "user", "content": [
    {"type": "text", "text": "test_parse_date_leap_year fails. Find out why and fix it."}
  ]},
  {"role": "assistant", "content": [
    {"type": "text", "text": "I'll run the failing test first."},
    {"type": "tool_use", "id": "toolu_01", "name": "Bash",
     "input": {"command": "python -m pytest tests/test_dates.py -k leap_year -q"}}
  ]},
  {"role": "user", "content": [
    {"type": "tool_result", "tool_use_id": "toolu_01",
     "content": "FAILED ... ValueError: day is out of range for month\n1 failed in 0.21s"}
  ]}
]
```

The next call sends all three messages, and the model's reply is the fourth. Nothing else is remembered. If the client dropped the third message, the model would not know the test had been run.

### Stop reasons

Each model reply also says why it stopped writing. The client branches on that value.

| Stop reason | Meaning | What the client does |
|---|---|---|
| `tool_use` | the reply ends with one or more tool calls | run them, append the results, call again |
| `end_turn` | the model finished its message | treat the text as the answer, unless a check sends it back |
| `max_tokens` | the reply hit its length cap mid-message | continue or fail, depending on the client |
| `stop_sequence` | a configured stop string appeared | client-specific |

A coding agent's loop is, at heart, "while the stop reason is `tool_use`, run the tools and call again".

## A minimal loop in code

The whole loop fits on one screen. This sketch uses the Anthropic Python SDK's shape; the model name and the single tool are placeholders, and a real client adds permissions, hooks, streaming, retries and output limits.

```python
import subprocess
import anthropic

client = anthropic.Anthropic()
TOOLS = [{
    "name": "run",
    "description": "Run a shell command in the project and return its output.",
    "input_schema": {"type": "object",
                     "properties": {"command": {"type": "string"}},
                     "required": ["command"]},
}]

def run_tool(name, args):
    if name != "run":
        return f"unknown tool {name}"
    done = subprocess.run(args["command"], shell=True, capture_output=True,
                          text=True, timeout=120)
    return (done.stdout + done.stderr)[-4000:]   # keep only the last 4,000 characters

def agent(task, max_turns=20):
    messages = [{"role": "user", "content": task}]
    for _ in range(max_turns):                    # an outside limit
        reply = client.messages.create(model="MODEL", max_tokens=4096,
                                       tools=TOOLS, messages=messages)
        messages.append({"role": "assistant", "content": reply.content})
        if reply.stop_reason != "tool_use":
            return reply.content                  # final answer
        results = [{"type": "tool_result", "tool_use_id": b.id,
                    "content": run_tool(b.name, b.input)}
                   for b in reply.content if b.type == "tool_use"]
        messages.append({"role": "user", "content": results})
    return "stopped: turn limit reached"
```

Three lines in that sketch are already harness decisions: the `max_turns` limit, the 4,000-character tail on tool output, and the 120-second command timeout. Every production client makes the same decisions, with more care.

### Worked example: where the sketch would go wrong

Give the sketch the task "delete the build directory and rebuild". It will run `rm -r build` without asking, because nothing between `reply` and `run_tool` checks anything. Claude Code puts a permission check and the `PreToolUse` hook exactly there, which is why it can ask before a deletion, or refuse one outright. The fix in the sketch is a single function call before `run_tool`, and that function is where most of a harness's safety lives.

## Parallel and sequential tool calls

A model can ask for several tools in one reply. Reading three files at once costs one round trip instead of three, and clients run independent calls in parallel. The model is told, in the client's system instructions, to batch calls that do not depend on each other, and to make dependent calls one at a time.

A turn with parallel calls, step by step:

1. The model writes one reply with three `tool_use` blocks: `Read(a.py)`, `Read(b.py)` and `Grep("parse_date")`.
2. The client runs the three calls at the same time.
3. The client sends one `user` message back with three `tool_result` blocks, each tagged with its call's id.
4. The model reads all three results and writes the next reply, here a single `Edit(a.py)` that depends on what it read.

| | Parallel calls | Sequential calls |
|---|---|---|
| Round trips | one for the whole batch | one per call |
| When the model decides | before any result is seen | after each result |
| Fits | independent reads and searches | a call whose input depends on an earlier result |

The trade-off is that parallel calls are decided together, before any result is seen. If the second call depends on what the first returns, it has to wait for the next turn.

## Where a harness attaches

Every part of a harness hooks into one of the loop's steps.

| Loop step | What the harness controls there | Covered in |
|---|---|---|
| Read the context | context files, memory, skills' descriptions, injected prompts | [Context files]({{ '/sections/04-context-files/' | relative_url }}), [Templates]({{ '/sections/07-templates-and-stubs/' | relative_url }}) |
| Choose an action | which tools exist, how they are described, which model runs | [Tools and tool calling]({{ '/sections/01-agentic-ai-fundamentals/tools.html' | relative_url }}), [Skills]({{ '/sections/05-skills-and-plugins/' | relative_url }}) |
| Run the tool | permissions, `PreToolUse` hooks, sandboxes, worktrees | [Permissions]({{ '/sections/01-agentic-ai-fundamentals/permissions-and-untrusted-input.html' | relative_url }}), [Hooks]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) |
| Add the result | `PostToolUse` hooks, output trimming, logging | [Hooks]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}), [Logging]({{ '/sections/16-logging-and-evaluation/' | relative_url }}) |
| Stop | completion checks, `Stop` hooks, `/goal`, limits | [How an agent run ends]({{ '/sections/01-agentic-ai-fundamentals/how-a-run-ends.html' | relative_url }}) |

### Worked example: one rule, attached at three steps

Take the rule "never commit with a failing test suite".

1. At the read step it is a line in `CLAUDE.md`. The model usually follows it.
2. At the run step it is a `PreToolUse` hook on `git commit` that runs the tests and refuses the commit on a failure. It runs on every commit attempt.
3. At the stop step it is a `Stop` hook that refuses to let the agent finish while the suite fails.
   - This covers a run where the agent never tried to commit at all, which the hook in step 2 never sees.

## Loops inside loops

A sub-agent is the same loop, started by a tool call in another loop.

![Sequence diagram: the parent model sends a tool_use Agent call with a brief to the client; the client starts the sub-agent model with the brief and context files; the sub-agent calls Grep and Read, the client runs them and the results return to the sub-agent; the sub-agent's final answer, a summary, goes back to the client, which returns only that summary to the parent as a tool result]({{ '/assets/images/01-agentic-ai-fundamentals/sub-agent-loop.svg' | relative_url }}){: style="width: 100%" }
*Figure: one delegation. Everything inside the middle band stays in the sub-agent's own context window.*

1. The parent's model calls a tool such as `Agent`, with a brief as its input.
2. The client starts a fresh loop for the sub-agent.
   - Its context holds the brief and the context files, and nothing from the parent's conversation.
3. The sub-agent runs its own turns: tool calls, results, more tool calls.
4. When the sub-agent writes a final answer, the client returns that answer to the parent as the `Agent` call's tool result.
   - The parent never sees the sub-agent's intermediate turns.

Delegation saves the parent's context because of step 4, and the same step is where detail is lost: the sub-agent's caveats survive only if its final answer states them. [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}) builds on this.

## Pitfalls

- Trusting the final message. The loop's last text is the model's own account of what it did. The tool calls and the files are the record; check those.
- Unbounded loops. A loop with no turn, time or cost limit runs until something else stops it.
- Untrimmed results. One verbose command can push the task's own instructions out of easy reach.
- A check at the wrong step. A rule enforced only in the context file is advice; the same rule enforced at the run step is a gate.

## Sources

- Tool use with the Messages API, content blocks and stop reasons: <https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview>, not yet verified against the primary source for this page's date.
- Hook events and where they fire: <https://code.claude.com/docs/en/hooks.md>, not yet verified against the primary source for this page's date.
- The code sketch, JSON excerpt and figure are illustrations written for this page.

{% include page_nav.html %}
