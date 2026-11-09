---
title: "Native commands and tools"
parent: "What is a harness?"
nav_order: 3
redirect_from:
  - /sections/02-ai-harnesses/native-commands-and-tools.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Native commands and tools

The basics of both tools are in [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}).

Native tools are what the model can call before you add anything. Native commands are what a person types. Together they are the floor of every harness: skills call the tools, hooks watch them, permission rules name them. This page lists them, shows how a tool call moves through the harness, and gives the checks to run before you build something that already exists.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## How a tool call works

### The loop

1. Each tool has a name, a description and an input schema, and all three sit in the context.
2. The model replies with a tool call in place of text.
3. The harness runs the `PreToolUse` hooks, then checks the permission rules, and asks the person if a rule says ask.
4. The tool runs, and the `PostToolUse` hooks see the result.
5. The result goes back to the model as the next message, and the loop continues.

```text
assistant  tool_use  Bash  {"command": "grep -rn 'def parse_date' src"}
harness    result    src/util/dates.py:14:def parse_date(value: str) -> date:
assistant  tool_use  Read  {"file_path": "src/util/dates.py", "offset": 10, "limit": 30}
```

![A sequence diagram of one bug-fix task in which the model makes tool calls through the harness: a refused Bash pattern kill, then Edit and a pytest run under an allow rule, a lint report from a PostToolUse hook, and a reviewer sub-agent started with the Agent tool.]({{ '/assets/images/02-ai-harnesses/harness-one-task.png' | relative_url }})

*Native tool calls in one real task, with the hooks and rules that wrap them. Our own diagram, made with archify.*

### Names matter

The tool names are the exact strings three other parts of the harness use:

- permission rules, such as `Bash(npm test *)` or `Read(./.env)`;
- hook matchers, such as `Bash` or `Edit|Write`;
- a sub-agent's `tools` list, such as `Read, Grep, Bash`.

A rule for `Bash` sees every shell command; a rule for `Edit` sees every file edit. The label a tool shows in the transcript can differ from its name, so check the [tools reference](https://code.claude.com/docs/en/tools-reference.md) when a rule does not match.

## Claude Code's tools

### Files and the shell

| Tool | Does | Asks by default |
|---|---|---|
| `Read` | reads a file or part of one, including images and PDFs | no, inside the working folders |
| `Write` | creates or overwrites a file | yes |
| `Edit` | replaces exact text in a file; the usual way to change code | yes |
| `NotebookEdit` | edits a Jupyter cell | yes |
| `Bash` | runs a shell command | yes, except a built-in set of read-only commands |
| `PowerShell` | runs PowerShell natively, where available | yes |
| `Glob`, `Grep` | find files by name, search contents | no; present by default on Windows only |
| `LSP` | jumps to definitions, finds references, reports type errors through a language server | no |

> On macOS, Linux and WSL, Claude Code leaves `Glob` and `Grep` out of the default tool set. Claude searches with `find` and `grep` through `Bash`, which run embedded versions of `bfs` and `ugrep`. Those searches reach your hooks and permission rules as `Bash` calls, so a `Bash` hook sees them.
{: .note }

### Web, agents and workflow

| Tool | Does | Asks by default |
|---|---|---|
| `WebFetch` | fetches a URL | yes, except preapproved documentation domains |
| `WebSearch` | searches the web | yes |
| `Agent` | starts a sub-agent with its own context window | no |
| `Skill` | runs a skill in the main conversation | yes |
| `TaskCreate`, `TaskList`, `TaskUpdate`, `TaskGet` | keep a task list for the session; they replace the older `TodoWrite` | no |
| `AskUserQuestion` | asks the person multiple-choice questions | no |
| `EnterPlanMode`, `ExitPlanMode` | switch into plan mode and present a plan | exit asks |
| `EnterWorktree`, `ExitWorktree` | move the session into an isolated git worktree and back | |
| `Monitor` | runs a command in the background and feeds each output line back | |
| `ToolSearch` | finds and loads deferred tools, such as MCP tools, when tool search is on | no |

MCP servers add more tools, named `mcp__<server>__<tool>`. See [Plugins and MCP servers]({{ '/sections/02-what-is-a-harness/plugins-and-mcp-servers.html' | relative_url }}).

## Codex's tools

Codex gives the model a smaller set and leans on the shell.

| Need | Claude Code | Codex |
|---|---|---|
| read and search files | `Read`, plus `find` and `grep` through `Bash` | shell commands such as `cat` and `rg` |
| edit files | `Edit`, `Write` | apply patch |
| run commands | `Bash` | shell |
| plan | `TaskCreate` and the other task tools, plan mode | plan tool |
| web | `WebFetch`, `WebSearch` | web search, when enabled |
| outside services | MCP tools | MCP tools from `[mcp_servers.<id>]` in `config.toml` |

Codex's `sandbox_mode` (`read-only`, `workspace-write`, `danger-full-access`) decides what the shell may write and reach on the network, and `approval_policy` decides when it asks. Together they do much of what Claude Code's per-tool permission rules do.

## Dedicated tool or shell

| Prefer | Over | Because |
|---|---|---|
| `Read` | `cat`, `head` | ranges, images, and reads inside the working folders need no prompt |
| `Edit` | `sed -i` | exact matches, clear diffs, and an `Edit` rule or hook sees it |
| `Bash` | a custom tool | tests, builds, git and your own scripts already run there |
| a script called from `Bash` | a long chain of shell commands | one command the agent cannot get half right, and one pattern for a rule to allow |

> A `Bash` permission rule matches the command text. Claude Code splits compound commands on `&&`, `;`, `|` and the rest, and each part must be allowed, but a rule for `curl *` does not match `/usr/bin/curl` or `sh -c 'curl ...'`. For filesystem and network limits that do not depend on the command text, use the sandbox; for full inspection, a [`PreToolUse` hook]({{ '/sections/02-what-is-a-harness/guard-hooks.html' | relative_url }}).
{: .warning }

## Commands the person types

### Session commands

| Command | Does |
|---|---|
| `/init` | drafts a `CLAUDE.md` for the project; offers to reuse Codex or Gemini CLI configuration it finds |
| `/clear` | starts a new conversation with an empty context |
| `/compact [instructions]` | summarises the conversation so far, focusing on what you name |
| `/context [all]` | shows context use as a grid, with suggestions for heavy tools and memory |
| `/resume` | reopens an earlier conversation |
| `/rewind` | rewinds the conversation, the code or both to an earlier point |
| `/model` | switches model and saves it as your default |
| `/usage` | shows session cost, plan limits and activity; `/cost` is an alias |

### Configuration commands

| Command | Does |
|---|---|
| `/memory` | edits `CLAUDE.md` files and turns auto memory on or off |
| `/permissions` | views and edits allow, ask and deny rules by scope |
| `/hooks` | shows hook configuration |
| `/skills` | lists skills; can sort them by token count |
| `/agents` | points you to the agent profile folders (older versions open an editor) |
| `/mcp` | manages MCP server connections and sign-in |
| `/plugin` | installs, enables and disables plugins |
| `/config` | opens settings; `/config key=value` sets one directly |
| `/status` | shows version, model, account and which settings files loaded |
| `/doctor` | checks the setup and proposes fixes; `/doctor prompt-audit` reviews instruction files |

Codex has its own set, including `/status` for the session's model and context, `/approvals` for its approval mode, `/compact`, `/init` and `/skills`.

### Skills as commands

A skill you write appears as `/name` next to the built-in commands.

- The person sees one menu; the harness knows which entries are built in and which are yours.
- A project skill named like a built-in command can replace it, so avoid the built-in names.
- Up to six skills can start one message: `/skill-a /skill-b do this`.
- See [Harness skills]({{ '/sections/02-what-is-a-harness/skills.html' | relative_url }}).

## Worked examples

### Before building a summariser

You want the agent to "summarise the session so far, keeping the open decisions". The native command already does it:

```text
> /compact keep the open decisions and the test commands
```

- It uses the harness's own summariser.
- The summary becomes the new context, and the project-root `CLAUDE.md` is re-read from disk.
- A `PreCompact` hook can check that state was written down first, and its exit code 2 blocks the compaction until it is.

### Before building a search helper

You want a tool that finds every caller of a function.

1. Ask for the search in plain words. Claude runs `grep` or `find` through `Bash`, and one call usually answers it.
2. If the search needs knowledge text search lacks, such as definitions and references across files, the `LSP` tool may already have it.
3. If it needs outside data, such as a database schema or a ticket system, that is a case for an MCP server.
4. If it is a fixed sequence of searches you run often, that is a case for a skill with a script.

### Measuring before trimming

```text
> /context all
```

At the start of a fresh session this shows the standing cost of the harness: system prompt, tools, MCP tools, memory files and skills. Run it first when a session feels crowded, and again after each change to the harness.

<!-- IMAGE NEEDED: a screenshot of Claude Code's /context grid at the start of a fresh session; search: claude code /context grid screenshot, claude code context usage visualization -->

## Pitfalls

| Pitfall | Result | Fix |
|---|---|---|
| a hook or rule written for `Grep` on Linux | it never fires; searches arrive as `Bash` | match `Bash` and inspect the command |
| a custom tool for what `Bash` already does | another description in the context on every turn | a script the agent runs through `Bash` |
| a `Bash` allow rule with arguments in the middle | it matches less than you expect, or more | put the `*` after the subcommand; use a hook for complex cases |
| relying on `/compact` alone before a long break | detail that was only in chat is gone | write a resume note first |

## Sources

- Claude Code tools reference: <https://code.claude.com/docs/en/tools-reference.md>, verified 2026-11-09.
- Claude Code commands: <https://code.claude.com/docs/en/commands.md>, verified 2026-11-09.
- Claude Code permissions and compound commands: <https://code.claude.com/docs/en/permissions.md>, verified 2026-11-09.
- Codex configuration: <https://learn.chatgpt.com/docs/config-file/config-reference>, verified 2026-11-09.

{% include page_nav.html %}
