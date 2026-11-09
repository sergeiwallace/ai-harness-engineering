---
title: "Tools and tool calling"
parent: "Agentic AI fundamentals"
nav_order: 2
has_children: true
has_toc: false
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Tools and tool calling

A tool is a named action the client can carry out for the model: read a file, run a command, search the web. Tools are what separate an agent from a chat model, and they are the largest surface a harness controls.

The [Agentic AI fundamentals overview]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}#tools) is the short version. This page covers:

- what a tool is made of, and the messages that carry a call and its result
- the built-in tools of Claude Code, and the settings that bound their output
- how to write a tool the model will pick correctly
- what tools cost in context, and where new tools come from.

Two sub-topic pages go further: [MCP servers]({{ '/sections/01-agentic-ai-fundamentals/mcp-servers.html' | relative_url }}) for adding tools from outside, and [Permissions and untrusted input]({{ '/sections/01-agentic-ai-fundamentals/permissions-and-untrusted-input.html' | relative_url }}) for controlling them.

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

The description is the only thing the model knows about the tool, and it is read on every turn. A vague description causes most wrong picks; [Writing a tool the model will use well](#writing-a-tool-the-model-will-use-well) shows how to fix one.

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

When a tool fails, the client still returns a `tool_result`, marked as an error, with the message:

```json
{"type": "tool_result", "tool_use_id": "toolu_08", "is_error": true,
 "content": "File not found: src/date.py"}
```

| Error the model reads | What it usually does next |
|---|---|
| file not found | searches for the right name, here `src/dates.py` |
| syntax error after an edit | reads the line and fixes it |
| command timed out | reruns with a narrower scope or a quiet flag |
| empty output, no error | guesses, because it has nothing to go on |

The last row is the case to design out: a tool should always say what happened.

## The built-in tools of a coding agent

### Claude Code's core set

Claude Code ships with a few dozen tools; the ones below cover most coding work. The tool names are the exact strings used in permission rules and hook matchers.

| Group | Tools | What they are for | Asks permission in Manual mode |
|---|---|---|---|
| Reading | `Read` | open a file, with line numbers; also images and PDFs | no, inside the working directory |
| Searching | `find` and `grep` through `Bash`; `Glob` and `Grep` on Windows | find files by name, search contents | no for the built-in read-only commands |
| Writing | `Edit`, `Write`, `NotebookEdit` | change part of a file, write a whole file, edit a notebook cell | yes |
| Running | `Bash` | any shell command: tests, builds, git | yes, except a built-in set of read-only commands |
| Web | `WebFetch`, `WebSearch` | fetch a page, search the web | yes, except some preapproved documentation sites for `WebFetch` |
| Work management | `Agent`, `TaskCreate` and related task tools, `Skill` | start a sub-agent, track steps, run a skill | varies |

Two details that surprise people:

- On macOS, Linux and WSL, Claude Code leaves `Glob` and `Grep` out of the default set. Claude searches with `find` and `grep` through `Bash`, which run built-in fast versions (`bfs` and `ugrep`), so the searches reach your hooks and permission rules as `Bash` calls.
- `Read` returns line numbers, which lets the model cite and edit exact lines.

Codex has its own set with the same shape: a shell, file edits applied as patches, and search.

### Limits on a tool's output

Claude Code bounds what a single `Bash` call can put into the context:

| Result | What reaches the model inline | Beyond that |
|---|---|---|
| Command succeeded | up to about 30,000 characters | the output is saved to a file; the model gets the path and a 2,000-character preview, and reads or searches the file if it needs more |
| Command failed | up to about 10,000 characters | a head-and-tail excerpt of the output |

The settings involved:

| Setting | Default | What it changes |
|---|---|---|
| `BASH_DEFAULT_TIMEOUT_MS` | 2 minutes | the timeout when the model asks for none |
| `BASH_MAX_TIMEOUT_MS` | 10 minutes | the longest timeout the model may ask for |
| `BASH_MAX_OUTPUT_LENGTH` | 30,000 characters | how much output is read back from the working file (up to 150,000) |
| `bashOutputMaxChars` (settings file) | not set | the inline limit and the read-back window together (up to 128,000) |

Raising these lets a verbose build reach the model whole, at the price of a larger context.

### Why narrow tools exist beside the shell

`Bash` can do everything the others do: `cat` reads, `sed` edits, `grep` searches. The narrow tools exist anyway, for three reasons.

#### They can be checked

An `Edit` call names one file and one change, so a rule on the path decides it exactly. A shell command is a string the client has to interpret.

| The same change to `build/app.js`, a generated file | Covered by `deny: Edit(build/**)`? |
|---|---|
| `Edit(build/app.js)` | yes |
| `Bash(sed -i 's/a/b/' build/app.js)` | yes: Claude Code recognises `sed`, `cat`, `tee` and similar file commands, and redirections such as `> build/app.js` |
| `Bash(python tools/regen.py)`, where the program writes `build/app.js` | no: the command does not name the file |

The last row needs a lower layer: a `PreToolUse` hook that knows which scripts write `build/`, or the sandbox, which can make `build/` read-only for every process.

<!-- IMAGE NEEDED: one edit attempted three ways (Edit tool, sed in the shell, a script that writes the file) against a deny rule on build/, showing the first two refused and the third passing; search: claude code permission rules sandbox diagram, agent tool call permission check -->

#### They show the change

The client can display an `Edit` as a diff for you to approve. A `sed` command shows only the command line until it has run.

#### They protect the window

`Read` returns line numbers and can read a range of lines, so the model takes only the part it needs. A shell `cat` returns the whole file, up to the output limit above.

## Writing a tool the model will use well

### The description does the work

A model picks a tool the way a new colleague picks a script from a folder: by reading what it says on the label. Four things make a description work.

1. Say what it returns, so the model knows what it will get back.
2. Say when to use it, especially relative to similar tools.
3. Say when not to use it.
4. Name the inputs plainly, with an example value where the format is not obvious.

Two schema details help too:

- mark only the inputs the tool cannot work without as `required`, because a model may guess a value for a required input you did not supply
- for a tool you define through the API, `strict: true` makes the model's calls match the schema exactly.

### Worked example: fixing a wrong pick

A project adds two tools, `get_logs` and `search_logs`. The agent keeps calling `get_logs`, which returns the whole day's log, and floods its window.

The descriptions were "Get logs" and "Search logs". Rewriting them fixes it:

| Before | After |
|---|---|
| `get_logs`: Get logs. | `get_logs`: Return the last N lines (default 50) of one service's log. Use for "what just happened". Do not use to find an old event; use `search_logs`. |
| `search_logs`: Search logs. | `search_logs`: Return log lines matching a pattern within a time range. Use to find a specific error or request id. |

### Shape the output for the model

Three rules for what a tool sends back:

| Rule | Weak output | Better output |
|---|---|---|
| Return less | every test line, all matches, the whole log | failures only, the first 50 matches, the last 100 log lines |
| Make errors actionable | `Permission denied` | `Permission denied writing build/: build/ is generated; edit src/ instead` |
| Keep the format stable | columns or wording that change between calls | the same fields in the same order every time |

#### Worked example: a log tool's defaults

```text
get_logs(service="api")              -> last 100 lines, newest last
get_logs(service="api", lines=500)   -> last 500 lines
get_logs(service="api", lines=50000) -> refused: "use search_logs for more than 1,000 lines"
```

The default answers "what just happened" in a few thousand tokens, and the cap stops one call from filling the window.

## What tools cost

### The fixed cost: definitions

Every tool loaded into the context is sent on every turn. Three things make up the fixed cost:

| Part | Size | Notes |
|---|---|---|
| The API's tool-use system prompt | about 286 tokens on current Claude models | added whenever any tool is defined |
| Built-in tool definitions | a few thousand tokens | written to be compact |
| Added tools: MCP servers, plugins | depends on the server | with tool search on, only names load until a tool is needed |

Claude Code turns MCP tool search on by default: MCP tool definitions are deferred, only tool names and server instructions load at session start, and the `ToolSearch` tool loads a definition when the model needs it. A client without tool search, or a session with it turned off (`ENABLE_TOOL_SEARCH=false`), loads every definition up front.

<details markdown="1">
<summary>A rough worked estimate, without tool search</summary>

Suppose each added tool's name, description and schema average 300 tokens.

| Setup | Fixed cost on every turn |
|---|---|
| 10 MCP servers, 8 tools each, without tool search | 80 tools, about 24,000 tokens: 12 percent of a 200,000-token window, under 3 percent of a one-million-token window |
| The same servers, with tool search on | roughly the 80 tool names, until one is used |

</details>

### The variable cost: results

Results are usually the larger cost, because they accumulate: every result stays in the history for the rest of the session.

- In Claude Code one `Bash` result reaches the model inline up to about 30,000 characters, roughly 7,500 tokens, and longer output goes to a file ([Limits on a tool's output](#limits-on-a-tools-output)).
- MCP tool output is limited to 25,000 tokens by default (`MAX_MCP_OUTPUT_TOKENS`), with a warning above 10,000.

| Call | What happens | Cheaper form |
|---|---|---|
| `cat` of a 3,000-line generated file | fills the inline limit, then the agent reads the saved file anyway | `Read` with a line range, or skip generated files |
| `git log` with no limit | fills the inline limit with old history | `git log --oneline -20` |
| a verbose test run, repeated ten times | about 75,000 tokens of mostly "passed" lines in the history | `pytest -q` |
| a search with no file filter on a large repository | thousands of matches, most irrelevant | a file filter and a match cap |

The token figures are estimates. [The context window and tokens]({{ '/sections/01-agentic-ai-fundamentals/context-window-and-tokens.html' | relative_url }}) works through a whole session's budget.

### The risk cost: reach

The more a tool can do, the more damage a wrong call can do.

| Tool | Worst plausible wrong call | Control that fits |
|---|---|---|
| `Read` | reads a secrets file into the context | deny rule on `.env` and key files |
| `Edit` | changes a generated or vendored file | deny rule on the path |
| `Bash` | deletes a directory, pushes to a shared branch | ask rules, a `PreToolUse` hook, a sandbox |
| an MCP write tool | changes records in an outside service | read-only credentials, ask rules |

These controls attach at the point where tools run; [Permissions and untrusted input]({{ '/sections/01-agentic-ai-fundamentals/permissions-and-untrusted-input.html' | relative_url }}) covers them.

## Where tools come from

| Source | What it adds | Covered in |
|---|---|---|
| Built-in | the core coding set | this page |
| MCP server | tools from an outside service, through a standard protocol | [MCP servers]({{ '/sections/01-agentic-ai-fundamentals/mcp-servers.html' | relative_url }}) |
| Plugin | a bundle that can include servers, skills and hooks | [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}) |
| Skill | instructions for combining existing tools, run through the built-in `Skill` tool, so no new tool entry is added | [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}) |

Choosing between them for a job such as "run these three commands in this order and check the output":

| Option | Fits when |
|---|---|
| A skill | the steps are a procedure the model follows, using the shell it already has |
| A script, called through `Bash` | the steps must run the same way every time; the script enforces the order |
| An MCP server | the job needs structured access to an outside service, or must work in clients without a shell |

The script is the enforcing option in the [systems approach]({{ '/' | relative_url }}#a-systems-approach): a skill describes the steps, and a script runs them.

## Pitfalls

- Adding a tool for every task. Each one costs context on every turn; prefer a skill or a script the shell can run.
- Vague descriptions. Two tools that sound alike lead to wrong picks.
- Unbounded output. A tool with no default limit floods the window on its first call.
- Silent failure. A tool that returns nothing on error leaves the model guessing.

## Sources

- Tool definitions, `stop_reason: "tool_use"`, `tool_result` blocks, parallel calls and the tool-use system prompt sizes: [Claude API docs: Tool use with Claude](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview), verified 2026-11-09.
- The `is_error` field on a failed result: [Claude API docs: Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls), not yet verified against the primary source for this page's date.
- Claude Code's built-in tools, the `Glob` and `Grep` defaults, `Bash` output limits and timeout variables: [Claude Code docs: Tools reference](https://code.claude.com/docs/en/tools-reference.md), verified 2026-11-09; the list changes between versions.
- MCP tool search and `MAX_MCP_OUTPUT_TOKENS`: [Claude Code docs: MCP](https://code.claude.com/docs/en/mcp.md), verified 2026-11-09.
- Read and Edit rule coverage of shell file commands: [Claude Code docs: Configure permissions](https://code.claude.com/docs/en/permissions.md), verified 2026-11-09.
- The tool definitions, log tool and examples are illustrations written for this page.

{% include page_nav.html %}
