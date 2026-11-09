---
title: "Plugins and MCP servers"
parent: "AI harnesses"
nav_order: 4
---

{% include page_nav.html %}

# Plugins and MCP servers

As of 2026-11-09. Part of [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}).

MCP servers connect a harness to services beyond the local machine. Plugins package skills, agent profiles, hooks and MCP servers so they install together. One extends what the agent can reach; the other makes a set of extensions shareable.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The Model Context Protocol

### What it is

The Model Context Protocol (MCP) is an open standard for connecting a model to outside services.

- Introduced by Anthropic in 2024; supported by Claude Code, Codex and many other clients.
- A server exposes tools, and optionally resources and prompts.
- The harness lists the server's tools next to its native ones, and the model calls them the same way.
- The protocol is JSON-RPC: the client sends `tools/list` to learn the tools and `tools/call` to run one.

### What a server offers

| Offers | Example |
|---|---|
| Tools | `search_docs(query)`, `create_issue(title, body)`, `send_message(to, text)` |
| Resources | a file, a database row, a page the model can read by reference |
| Prompts | a reusable prompt the person can pick from a menu |

### Transports

| Transport | Runs | Good for |
|---|---|---|
| stdio | as a local process the harness starts | local tools, files, databases |
| streamable HTTP | as a remote service | hosted APIs, shared team services |
| SSE | as a remote service | older servers; deprecated in Claude Code |

![A sequence diagram of one MCP tool call. At session start the harness gives the model a tool list that includes mcp__docs__search; each schema costs tokens. The model calls mcp__docs__search with a query. The harness sends PreToolUse to the hooks, which allow it. The harness sends tools/call to the MCP server, which sends an API request to the outside service and gets results. The server returns text content to the harness, which returns the tool result to the model, to be treated as untrusted input.]({{ '/assets/images/02-ai-harnesses/mcp-call.png' | relative_url }})

*One MCP call, passing the same hooks as a native tool. Our own diagram, made with archify.*

## Configuring a server

### Claude Code

```text
$ claude mcp add --transport http --scope project docs-search https://mcp.example.com/docs
$ claude mcp add --env NOTES_DIR=/srv/notes notes -- python3 tools/notes_server.py
```

For a stdio server, everything after `--` is the command that starts it; the options before `--` belong to Claude Code.

| Scope | Loads in | Stored in | Shared |
|---|---|---|---|
| local (the default) | this project, for you | `~/.claude.json` | no |
| project | this project, for everyone | `.mcp.json` at the project root | yes, through git |
| user | all your projects | `~/.claude.json` | no |

At project scope the servers are written to `.mcp.json`, committed so the team shares them:

```json
{
  "mcpServers": {
    "docs-search": { "type": "http", "url": "https://mcp.example.com/docs" },
    "notes": { "command": "python3", "args": ["tools/notes_server.py"] }
  }
}
```

### Codex

```toml
[mcp_servers.notes]
command = "python3"
args = ["tools/notes_server.py"]
env = { NOTES_DIR = "/srv/notes" }

[mcp_servers.docs-search]
url = "https://mcp.example.com/docs"
```

### Managing servers

| Command | Does |
|---|---|
| `/mcp` | lists servers and their status; reconnects, enables, disables and signs in |
| `claude mcp list` | lists configured servers from the shell |
| `claude mcp reset-project-choices` | clears your approvals of project servers |

> Claude Code asks for approval before it uses servers from a project's `.mcp.json` for the first time. A cloned repository cannot start a server on your machine without that approval.
{: .note }

### Naming and permissions

MCP tools appear to the model and to your rules as `mcp__<server>__<tool>`. A server that comes from a plugin is named `mcp__plugin_<plugin>_<server>__<tool>`. Permission rules and hook matchers write patterns differently.

| Where | Pattern | Matches |
|---|---|---|
| permission rule | `mcp__notes__send` | one tool |
| permission rule | `mcp__notes__*` | every tool from one server |
| deny or ask rule | `mcp__*` | every MCP tool |
| hook matcher | `mcp__notes__.*` | every tool from one server (a regular expression) |
| hook matcher | `mcp__notes` | nothing: a bare server name matches no tool |

A guard written for `Bash` does not see MCP calls. If an MCP tool can delete, send or spend, give it its own permission rule or hook.

## What MCP costs

MCP has two costs: tool definitions in the context, and tool results.

| Cost | Default behaviour in Claude Code | Setting |
|---|---|---|
| tool definitions | tool search is on: definitions load when a search needs them, through `ToolSearch` | `ENABLE_TOOL_SEARCH=false` turns it off |
| a large tool result | a warning above 10,000 tokens; output capped at 25,000 tokens | `MAX_MCP_OUTPUT_TOKENS` |
| a very large text result | saved to a file in place of the context | the server can raise a tool's own limit |

Without tool search, a server with thirty tools can cost more than your whole context file. Connect only the servers a project needs either way.

> Claude Code's docs: "Verify you trust each server before connecting it. Servers that fetch external content can expose you to prompt injection risk." A server runs with your credentials; treat everything it returns as untrusted input.

<!-- IMAGE NEEDED: a prompt injection reaching the agent through an MCP tool result (outside page, MCP server, tool result, agent); search: MCP prompt injection diagram, tool poisoning MCP, indirect prompt injection agent diagram -->
{: .warning }

## Plugins

### What a plugin contains

```text
my-plugin/
  .claude-plugin/plugin.json   the manifest: name, plus version and description
  skills/                      one folder per skill, each with SKILL.md
  agents/                      agent profiles
  commands/                    command files (merged into skills)
  hooks/hooks.json             hooks for lifecycle events
  .mcp.json                    MCP servers
```

- A plugin skill runs as `/<plugin>:<skill>`, so two plugins can ship skills with the same name.
- While you build one, load it straight from its folder with `claude --plugin-dir ./my-plugin`.

<!-- IMAGE NEEDED: a plugin folder on the left (manifest, skill, agent, hooks, MCP server) and what each becomes in a session on the right; search: claude code plugin directory structure diagram, claude code plugin components -->

### Installing

```text
> /plugin marketplace add <owner>/<repo>
> /plugin install review-kit@<marketplace>
```

1. A marketplace is a repository or folder with a `.claude-plugin/marketplace.json` that lists plugins and where to fetch each one.
2. Claude Code adds Anthropic's official marketplace the first time you start an interactive session; others you add yourself.
3. You pick a scope when you install: user (every project), project (shared through `.claude/settings.json`) or local (you, this project).
4. `/plugin` lists what is installed and lets you disable a plugin without uninstalling it.

### What an enabled plugin costs

| Cost | When |
|---|---|
| the name and description of each skill, agent and command it adds | every turn of every session, even when nothing from it runs |
| the full skill or agent text | only when used |
| its MCP servers and hooks | running alongside every session where it is enabled |
| its permissions | whatever it runs, it runs as you |

### When to make one

| Situation | Use |
|---|---|
| your own machine, a few skills and hooks | loose files in `.claude/` |
| handing a working set to someone else in one step | a plugin |
| a skill that needs an MCP server and a guard to be safe | a plugin, so all three install together |
| one prompt you reuse | a custom command |

## Worked examples

### Messaging between sessions

Several sessions on one machine need to coordinate.

1. A small MCP server wraps a mailbox and gives each session `send`, `fetch` and `acknowledge` tools.
2. A skill explains when to message, how to word a hand-off, and when to wait for a reply.
3. Packaged as a plugin, a new machine gets the server and the skill in one install.

My harness runs a setup of this shape as a fallback for when the harness's built-in messaging cannot reach a session.

### A web search server

On a machine where the native web search tool is unavailable, an MCP server that calls an approved search service fills the gap.

- The context file names the server, so the agent uses it.
- It also says that queries leave the machine, so the agent puts no secrets in a query.
- A hook on `mcp__search__.*` can scan the query for credential patterns and refuse, which enforces the second point in code.

## Sources

- Model Context Protocol: <https://modelcontextprotocol.io/>.
- Claude Code MCP and plugins: <https://code.claude.com/docs/en/mcp.md>, <https://code.claude.com/docs/en/plugins.md>, verified 2026-11-09.
- Claude Code permission rules for MCP tools: <https://code.claude.com/docs/en/permissions.md>, verified 2026-11-09.
- Hook matchers for MCP tools: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-09.
- Codex MCP configuration: <https://learn.chatgpt.com/docs/config-file/config-reference>, verified 2026-11-09.
- The messaging and search examples are from the author's own harness, described conceptually.

{% include page_nav.html %}
