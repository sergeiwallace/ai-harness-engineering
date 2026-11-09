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
| HTTP | as a remote service | hosted APIs, shared team services |

![A sequence diagram of one MCP tool call. At session start the harness gives the model a tool list that includes mcp__docs__search; each schema costs tokens. The model calls mcp__docs__search with a query. The harness sends PreToolUse to the hooks, which allow it. The harness sends tools/call to the MCP server, which sends an API request to the outside service and gets results. The server returns text content to the harness, which returns the tool result to the model, to be treated as untrusted input.]({{ '/assets/images/02-ai-harnesses/mcp-call.png' | relative_url }})

*One MCP call, passing the same hooks as a native tool. Our own diagram, made with archify.*

## Configuring a server

### Claude Code

```text
claude mcp add --transport http docs-search https://mcp.example.com/docs
claude mcp add notes -- python3 tools/notes_server.py
```

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
```

### Naming and permissions

MCP tools appear to the model and to your rules as `mcp__<server>__<tool>`.

| Rule or matcher | Matches |
|---|---|
| `mcp__notes__send` | one tool |
| `mcp__notes__.*` (hook matcher) | every tool from one server |
| `mcp__.*` (hook matcher) | every MCP tool |

A guard written for `Bash` does not see MCP calls. If an MCP tool can delete, send or spend, give it its own permission rule or hook.

## What MCP costs

Every tool's name, description and schema sits in the context like a native tool's. A server with thirty tools can cost more than your whole context file. Connect the servers a project needs, not every server you have. Claude Code can defer loading tool definitions until they are searched for, which lowers the standing cost when many servers are connected.

> An MCP server runs with your credentials and can return any text into the context. Install servers from sources you trust, and treat their output as untrusted input.
{: .warning }

## Plugins

### What a plugin contains

```text
my-plugin/
  .claude-plugin/plugin.json   name, version, description
  skills/                      skills
  agents/                      agent profiles
  commands/                    custom commands
  hooks/                       hook definitions
  .mcp.json                    MCP servers
```

### Installing

```text
> /plugin marketplace add <owner>/<repo>
> /plugin install review-kit@<marketplace>
```

A marketplace is a repository that lists plugins. Installing one copies in every part it contains, at the version it names.

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
- Claude Code MCP and plugins: <https://code.claude.com/docs/en/mcp.md>, <https://code.claude.com/docs/en/plugins.md>.
- Codex MCP configuration: <https://developers.openai.com/codex/mcp>.
- The messaging and search examples are from the author's own harness, described conceptually.

{% include page_nav.html %}
