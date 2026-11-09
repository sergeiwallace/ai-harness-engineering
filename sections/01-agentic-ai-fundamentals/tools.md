---
title: "Tools and tool calling"
parent: "Agentic AI fundamentals"
nav_order: 2
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Tools and tool calling

As of 2026-11-09.

A tool is a named action the client can carry out for the model: read a file, run a command, search the web. Tools are what separate an agent from a chat model, and they are also the largest surface a harness controls. This page covers what a tool is made of, the built-in tools of a coding agent, how to write a tool the model will use well, and what tools cost. Two sub-topic pages go further: [MCP servers]({{ '/sections/01-agentic-ai-fundamentals/mcp-servers.html' | relative_url }}) for adding tools from outside, and [Permissions and untrusted input]({{ '/sections/01-agentic-ai-fundamentals/permissions-and-untrusted-input.html' | relative_url }}) for controlling them. The [Agentic AI fundamentals overview]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}#tools) is the short version.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Anatomy of a tool

### The three parts

Every tool the model sees has the same three parts.

| Part | Purpose | Read by |
|---|---|---|
| Name | a short identifier the model writes in its call | the model and the client |
| Description | prose saying what the tool does, when to use it and when not to | the model only |
| Input schema | a JSON Schema for the arguments | the model, to shape its call, and the client, to validate it |

The description is the part people underrate. It is the only thing the model knows about the tool, and it is read on every turn.

### Definition, call and result

A tool passes through three forms in a session:

1. The definition sits in the model's context on every turn.
2. When the model wants the tool, it writes a `tool_use` block against that definition, with an id and the input.
3. The client runs the call, after its permission checks, and returns a `tool_result` with the same id.

The three forms for one search tool:

```json
{"name": "search_code",
 "description": "Search the repository for a regular expression. Returns matching lines with file and line number. Use this before reading files when you do not know where something is. Do not use it for reading a whole file.",
 "input_schema": {"type": "object",
   "properties": {"pattern": {"type": "string"},
                  "glob": {"type": "string", "description": "optional file filter such as *.py"}},
   "required": ["pattern"]}}
```

```json
{"type": "tool_use", "id": "toolu_07", "name": "search_code",
 "input": {"pattern": "def parse_date", "glob": "*.py"}}
```

```json
{"type": "tool_result", "tool_use_id": "toolu_07",
 "content": "src/dates.py:41:def parse_date(text: str) -> date:"}
```

### Errors are results too

When a tool fails, the client still returns a `tool_result`, marked as an error, with the message. The model reads it and usually adapts: a missing file leads to a search, a syntax error leads to a fix. A tool that fails silently, returning nothing, gives the model nothing to adapt to.

## The built-in tools of a coding agent

### Claude Code's core set

Claude Code ships with a set of tools that covers most coding work. The exact list changes between versions, and the grouping below holds across them.

| Group | Tools | What they are for |
|---|---|---|
| Reading | `Read`, `Glob`, `Grep` | open a file, find files by pattern, search contents |
| Writing | `Edit`, `Write`, `NotebookEdit` | change part of a file, write a whole file, edit a notebook cell |
| Running | `Bash` | any shell command: tests, builds, git |
| Web | `WebFetch`, `WebSearch` | fetch a page, search the web |
| Work management | `Agent`, a task or to-do list tool | start a sub-agent, track steps |

Codex has its own set with the same shape: a shell, file edits applied as patches, and search.

### Why narrow tools exist beside the shell

`Bash` can do everything the others do: `cat` reads, `sed` edits, `grep` searches. The narrow tools exist anyway, for three reasons.

#### They can be checked

An `Edit` call names one file and one change. A permission rule or hook can allow edits to `src/` and refuse edits to `build/`. A shell command that does the same edit is a string that has to be parsed to know what it touches.

![Sequence diagram: the model asks to Edit build/app.js; the client checks the tool and path against the permission rules, a deny rule for build matches, and the call is refused, so the model edits the source file instead. Then the model asks the shell to run sed on build/app.js; the client checks only the command text, no Edit rule applies, sed rewrites the generated file and the result returns as done]({{ '/assets/images/01-agentic-ai-fundamentals/narrow-tool-vs-shell.svg' | relative_url }}){: style="width: 100%" }
*Figure: the same change to a generated file, refused as an `Edit` and carried out as a shell command. Closing the second path takes a hook that parses shell commands, or a sandbox that makes `build/` read-only.*

#### They show the change

The client can display an `Edit` as a diff for you to approve. A `sed` command is opaque until it has run.

#### They protect the window

`Read` can return a slice of a file with line numbers, and `Grep` can cap its matches. A shell `cat` returns everything.

> Prefer the narrowest tool that does the job. A permission rule or hook can check a narrow tool's call by its arguments, and its output is usually smaller.
{: style="border-left: 4px solid #0f766e; background: #f0fdfa; padding: 0.75rem 1rem;"}

## Writing a tool the model will use well

### The description does the work

A model picks a tool the way a new colleague picks a script from a folder: by reading what it says on the label. Four things make a description work.

1. Say what it returns, so the model knows what it will get back.
2. Say when to use it, especially relative to similar tools.
3. Say when not to use it.
4. Name the inputs plainly, with an example value where the format is not obvious.

### Worked example: fixing a wrong pick

A project adds two tools, `get_logs` and `search_logs`. The agent keeps calling `get_logs`, which returns the whole day's log, and floods its window. The descriptions were "Get logs" and "Search logs". Rewriting them fixes it:

| Before | After |
|---|---|
| `get_logs`: Get logs. | `get_logs`: Return the last N lines (default 50) of one service's log. Use for "what just happened". Do not use to find an old event; use `search_logs`. |
| `search_logs`: Search logs. | `search_logs`: Return log lines matching a pattern within a time range. Use to find a specific error or request id. |

### Shape the output for the model

#### Return less

Return only the lines that answer the question: a test runner that prints only failures, a search that caps at fifty matches, a log tool that defaults to the tail.

#### Make errors actionable

"Permission denied" tells the model nothing to do. "Permission denied writing build/: build/ is generated; edit src/ instead" tells it exactly what to do next.

#### Keep the format stable

The model learns a tool's output shape within a session. Changing column order or wording between calls makes it misread results.

## What tools cost

### The fixed cost: definitions

Every tool definition sits in the context on every turn. A built-in set is designed to be compact; a long list of added tools is not.

<details markdown="1">
<summary>A rough worked estimate</summary>

Suppose each added tool's name, description and schema average 300 tokens. Ten MCP servers with eight tools each is 80 tools, or about 24,000 tokens, present before the task starts and re-read on every turn. On a 200,000-token window that is 12 percent gone; on a one-million-token window it is under 3 percent, but it still costs attention and, on metered billing, money on every call that is not served from cache. Clients reduce this by loading tool definitions on demand rather than all at once; check your client's current behaviour before assuming either way.
</details>

### The variable cost: results

Results are usually the larger cost. One `cat` of a generated file, one unfiltered `git log`, one test run with verbose output can each add tens of thousands of tokens. [The context window and tokens]({{ '/sections/01-agentic-ai-fundamentals/context-window-and-tokens.html' | relative_url }}) works through a budget.

### The risk cost: reach

The more a tool can do, the more damage a wrong call can do. A shell can delete a directory or push to a shared branch. That is why permissions, hooks and sandboxes attach at the point where tools run; [Permissions and untrusted input]({{ '/sections/01-agentic-ai-fundamentals/permissions-and-untrusted-input.html' | relative_url }}) covers them.

## Where tools come from

| Source | What it adds | Covered in |
|---|---|---|
| Built-in | the core coding set | this page |
| MCP server | tools from an outside service, through a standard protocol | [MCP servers]({{ '/sections/01-agentic-ai-fundamentals/mcp-servers.html' | relative_url }}) |
| Plugin | a bundle that can include servers, skills and hooks | [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}) |
| Skill | not a tool itself, but instructions that tell the model how to combine tools for a task | [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}) |

A skill is often a better answer than a new tool. If the job is "run these three commands in this order and check this output", a skill written in prose uses the existing shell; a new tool would add a definition to every turn for a job done once a week.

## Pitfalls

- Adding a tool for every task. Each one costs context on every turn; prefer a skill or a script the shell can run.
- Vague descriptions. Two tools that sound alike lead to wrong picks.
- Unbounded output. A tool with no default limit floods the window on its first call.
- Silent failure. A tool that returns nothing on error leaves the model guessing.

## Sources

- Tool definitions, `tool_use` and `tool_result` blocks: <https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview>, not yet verified against the primary source for this page's date.
- Claude Code's built-in tools: <https://code.claude.com/docs/en/settings.md>, not yet verified against the primary source for this page's date; the list changes between versions.
- The definitions, examples and estimate are illustrations written for this page.

{% include page_nav.html %}
