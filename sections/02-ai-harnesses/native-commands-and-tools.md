---
title: "Native commands and tools"
parent: "AI harnesses"
nav_order: 3
---

{% include page_nav.html %}

# Native commands and tools

As of 2026-11-09. Part of [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}). The basics of both tools are in [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}).

Native tools are what the model can call before you add anything. Native commands are what a person can type. Together they are the floor of every harness: skills call the tools, hooks watch them, permission rules name them. Knowing them well saves you from building what already exists.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Tools the model calls

### How a tool call works

Each tool has a name, a description and an input schema, all of which sit in the context. The model replies with a tool call instead of text; the harness checks permissions and hooks, runs the tool, and returns the result as the next message.

```text
assistant  tool_use  Grep  {"pattern": "def parse_date", "path": "src"}
harness    result    src/util/dates.py:14:def parse_date(value: str) -> date:
assistant  tool_use  Read  {"file_path": "src/util/dates.py", "offset": 10, "limit": 30}
```

![A sequence diagram of one bug-fix task in which the model makes tool calls through the harness: a refused Bash pattern kill, then Edit and a pytest run under an allow rule, a lint report from a PostToolUse hook, and a reviewer sub-agent started with the Agent tool.]({{ '/assets/images/02-ai-harnesses/harness-one-task.png' | relative_url }})

*Native tool calls in one real task, with the hooks and rules that wrap them. Our own diagram, made with archify.*

### Claude Code's tools

| Tool | Does | Notes |
|---|---|---|
| Read | reads a file, or part of one | also images and PDFs |
| Write | creates or replaces a file | must read an existing file first |
| Edit | replaces an exact string in a file | the usual way to change code |
| Glob | finds files by name pattern | fast, no shell needed |
| Grep | searches file contents | wraps ripgrep |
| Bash | runs a shell command | can run anything the shell can, so it is the most guarded |
| WebFetch, WebSearch | reads a page, searches the web | subject to permission rules |
| Agent (Task) | starts a sub-agent with a brief | see [Agent profiles]({{ '/sections/02-ai-harnesses/agent-profiles.html' | relative_url }}) |
| TodoWrite | keeps a task list in the session | visible to the person |
| NotebookEdit | edits a Jupyter cell | |

### Codex's tools

Codex gives the model a smaller set and leans on the shell.

| Need | Claude Code | Codex |
|---|---|---|
| read and search files | Read, Grep, Glob | shell commands (`cat`, `rg`) |
| edit files | Edit, Write | apply patch |
| run commands | Bash | shell |
| plan | TodoWrite | plan tool |
| web | WebFetch, WebSearch | web search, when enabled |
| outside services | MCP tools | MCP tools |

Codex's sandbox decides what the shell may read, write and reach on the network, so it does much of the work Claude Code's per-tool permission rules do.

### Dedicated tool or shell

| Prefer | Over | Because |
|---|---|---|
| Read | `cat`, `head` | ranges, images, and the harness tracks what was read |
| Edit | `sed -i` | exact matches, clear diffs, fewer accidents |
| Grep, Glob | `grep -r`, `find` | faster, and permission rules can allow them broadly |
| Bash | anything else | tests, builds, git, your own scripts |

A permission rule or a hook can treat a dedicated tool very differently from a shell command. Read is safe to allow everywhere; `Bash(cat:*)` is a pattern that a clever command can step around.

## Commands the person types

### Session commands

| Command | Does |
|---|---|
| `/init` | drafts a `CLAUDE.md` from the repository |
| `/clear` | starts a fresh context in the same session |
| `/compact [instructions]` | summarises the context, keeping what you name |
| `/context` | shows how the window is used |
| `/resume` | reopens an earlier session |
| `/model` | switches model |
| `/cost` | shows token use and spend for the session |

### Configuration commands

| Command | Does |
|---|---|
| `/memory` | opens the context and memory files for editing |
| `/permissions` | views and edits allow, ask and deny rules |
| `/hooks` | lists loaded hooks |
| `/agents` | lists and edits agent profiles |
| `/mcp` | lists MCP servers and their status |
| `/plugin` | installs and manages plugins |
| `/config` | opens settings |

Codex has its own set, including `/status` for the session's model, limits and context, `/approvals` for its approval mode, `/compact` and `/init`.

### Skills as commands

A skill you write appears as `/name` next to the built-in commands.

- The person sees one menu; the harness knows which entries are native and which are yours.
- A name that clashes with a built-in command is confusing; prefix your own if needed.
- See [Harness skills]({{ '/sections/02-ai-harnesses/skills.html' | relative_url }}).

## Worked examples

### Before building a summariser

You want the agent to "summarise the session so far, keeping the open decisions". The native command already does it:

```text
> /compact keep the open decisions and the test commands
```

- It uses the harness's own summariser.
- The summary becomes the new context, and the context files reload.
- A `PreCompact` hook can check that state was written down first.

### Before building a search helper

You want a tool that finds every caller of a function.

1. Try Grep with the function name and a path filter. One call usually answers it.
2. If the search needs knowledge Grep lacks, such as a type checker's call graph or a database schema, that is a good case for an MCP server.
3. If it is a fixed sequence of searches you run often, it is a good case for a skill with a script.

### Measuring before trimming

```text
> /context
```

At the start of a fresh session this shows the standing cost of the harness: system prompt, tool descriptions, MCP tools, context files and skill summaries. It is the first thing to run when a session feels crowded.

## Sources

- Claude Code tools and slash commands: <https://code.claude.com/docs/en/settings.md#tools-available-to-claude>, <https://code.claude.com/docs/en/slash-commands.md>.
- Codex CLI: <https://developers.openai.com/codex/cli>.

{% include page_nav.html %}
