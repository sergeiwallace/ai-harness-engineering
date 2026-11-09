---
title: "MCP servers"
parent: "Tools and tool calling"
grand_parent: "Agentic AI fundamentals"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# MCP servers

The Model Context Protocol (MCP) is an open standard for plugging an outside service into an AI client as a set of tools. One server can serve Claude Code, Codex, a desktop chat app and an IDE, so a tool written once works in all of them. This page is part of [Tools and tool calling]({{ '/sections/01-agentic-ai-fundamentals/tools.html' | relative_url }}). It covers:

- how hosts, clients and servers fit together, and the two transports;
- adding a server from the command line or a project file, and the three scopes;
- what a server costs in context, trust and reliability;
- when a command-line tool or a skill is the better choice.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The shape of MCP

### Hosts, clients and servers

| Role | What it is | Example |
|---|---|---|
| Host | the application the person uses | Claude Code |
| Client | the connector inside the host, one per server | started by the host from its configuration |
| Server | a program that exposes capabilities | a server wrapping a database |

Clients and servers exchange JSON-RPC 2.0 messages. The model never talks to a server directly: the server's tools appear to the model like built-in ones, and when the model calls one, the host's client forwards the call.

### One server in a session

![Sequence diagram: at startup the host client starts and initializes the MCP server, asks it for its tools, receives names, descriptions and schemas, and puts the tool definitions in the model's context; later the model writes a tool_use for run_query, the host sends tools/call over JSON-RPC after its permission checks and hooks, the server runs a SELECT as a read-only database user, and the rows flow back as the result and then as a tool_result to the model]({{ '/assets/images/01-agentic-ai-fundamentals/mcp-startup-and-call.svg' | relative_url }}){: style="width: 100%" }
*Figure: one MCP server from start-up to a single call. The red arrow is where the server's own limits apply. With tool search on, the model sees the tool names at start-up and loads a full definition only when it needs one.*

1. At start-up the host starts each configured server and asks it for its tools.
2. The server answers with each tool's name, description and input schema.
3. The host makes the tools available to the model.
   - With tool search on, which is Claude Code's default, only the tool names and the server's instructions go into the context. The model loads a full definition through the `ToolSearch` tool when it needs one.
   - With tool search off, every definition goes into the context and stays there on every turn.
4. When the model calls a tool, the host runs its permission checks and hooks, then forwards the call to the server.
5. The server does the work and returns the result, which the host passes to the model as a `tool_result`.

### Transports

| | stdio | HTTP |
|---|---|---|
| How it runs | the host starts the server as a child process | the server runs as a web service |
| How they talk | standard input and output | HTTP, with streaming for long responses |
| Network port | none | yes |
| Authentication | runs as your local user | needed: a header token, or OAuth through `/mcp` |
| Fits | one developer's local tools | shared or hosted servers |

### What a server can offer

| Primitive | Controlled by | What it is |
|---|---|---|
| Tools | the model | actions the model can call, with a schema, like built-in tools |
| Resources | the application or person | data the host can attach to the context, such as a file or a record |
| Prompts | the person | reusable prompt templates, often shown as slash commands |

Most servers used in coding offer tools only.

## Adding a server

### From the command line

`claude mcp add` registers a server. Options come before the name; for a local server, `--` separates them from the command that starts it.

```bash
# a local server, started as a child process, with one environment variable
claude mcp add --transport stdio --env DB_URL="$DB_URL" my-db -- npx -y example-db-mcp-server --readonly

# a remote server over HTTP, with a token header
claude mcp add --transport http tracker https://mcp.example.com/mcp \
  --header "Authorization: Bearer $TRACKER_TOKEN"

# list servers and whether each connected
claude mcp list
```

### In a project file

A project can commit its servers so that everyone who opens it gets the same tools. Claude Code reads `.mcp.json` at the project root:

```json
{
  "mcpServers": {
    "my-db": {
      "command": "npx",
      "args": ["-y", "example-db-mcp-server", "--readonly"],
      "env": { "DB_URL": "${DB_URL}", "DB_POOL": "${DB_POOL:-2}" }
    }
  }
}
```

| Reference | Expands to |
|---|---|
| `${DB_URL}` | the value of `DB_URL` in your environment, so the secret stays out of the file |
| `${DB_POOL:-2}` | the value of `DB_POOL` if set, otherwise `2` |

If a referenced variable is unset and has no default, Claude Code warns and passes the literal `${...}` text.

### Scopes

| Scope | Stored in | Loads in | Shared? |
|---|---|---|---|
| Local (the default) | `~/.claude.json`, under this project | this project | no |
| Project | `.mcp.json` in the repository | this project | yes, through version control |
| User | `~/.claude.json` | all your projects | no |

A project-scoped server is a program a cloned repository can ask to run:

- in an interactive session, Claude Code asks for approval before using a project-scoped server for the first time; `claude mcp reset-project-choices` clears those answers;
- in `claude -p` runs and Agent SDK sessions there is no one to ask, so project-scoped servers load without asking. Exclude them there with `--strict-mcp-config` or the `disabledMcpjsonServers` setting.

### Worked example: a read-only database tool

A team wants the agent to answer "how many orders failed yesterday" without anyone pasting query results.

1. They add a database MCP server at project scope, started with `--readonly`.
2. The agent can now list tables, read a schema and run a `SELECT`.
3. Two layers keep it safe, both enforced below the model:
   - the server's read-only mode refuses any write statement;
   - the database user's grants allow only reads.
4. If the model is ever talked into writing, the server and the database both refuse, whatever the model was told.

## What a server costs

### Context

| Cost | With tool search on | With it off |
|---|---|---|
| At start-up | tool names and server instructions | every tool definition |
| On each turn | names, plus definitions the model has loaded | every definition |
| Per call | the result, up to `MAX_MCP_OUTPUT_TOKENS` (25,000 by default; warning above 10,000) | the same |

Ways to keep the cost down:

- disconnect servers a project does not use; `/mcp` can disable a server without removing it;
- keep tool descriptions short, with the words the model would search for near the start;
- prefer tools that return a page of results with a way to ask for more.

[Tools and tool calling]({{ '/sections/01-agentic-ai-fundamentals/tools.html' | relative_url }}#what-tools-cost) has a worked estimate.

### Trust

A server runs with your permissions, or is a remote service that receives your data. Its tool descriptions and results go into the model's context, where text can steer the model. A malicious or compromised server can therefore both act and inject instructions.

Treat a server like a dependency you install:

1. check who maintains it and how widely it is used;
2. pin its version in the configuration;
3. give it the narrowest credentials that work, such as a read-only token;
4. put ask rules on its write tools: `mcp__<server>__<tool>` names one tool, and `mcp__<server>__*` covers every tool from that server.

### Reliability

A local server that fails to start, or a remote one that times out, takes its tools with it.

```bash
claude mcp list    # from the shell: each server and whether it connected
/mcp               # inside a session: status, tool counts, reconnect, disable, re-authenticate
```

## When not to use a server

A command-line tool the shell already runs is often enough:

- it costs nothing in context until it is used;
- its output can be trimmed with ordinary flags;
- its permissions are the `Bash` rules you already manage.

| Need | Better fit |
|---|---|
| Call a service that has a solid command-line tool | the shell, optionally wrapped in a skill |
| A procedure of several steps | a skill, or a script the skill runs |
| The same tool in a chat app, an IDE and a terminal agent | an MCP server |
| Structured access with credentials the agent should not see | an MCP server |

## Pitfalls

- Project-scoped servers in headless runs load without a prompt; a CI job on an untrusted branch can start whatever `.mcp.json` names.
- A secret pasted into `.mcp.json` is committed with it; use `${VAR}` references.
- A tool that returns everything at once fills the output limit and leaves the model to page through a saved file.
- Write tools with no ask rule: an injected instruction in a fetched page can reach them.

## Sources

- Claude Code MCP: `claude mcp add` syntax, scopes and storage, `.mcp.json` variable expansion, project-server approval, `/mcp`, tool search and `MAX_MCP_OUTPUT_TOKENS`: [Claude Code docs: MCP](https://code.claude.com/docs/en/mcp.md), verified 2026-11-09.
- The Model Context Protocol specification, transports and primitives: [Model Context Protocol specification](https://modelcontextprotocol.io/specification), not yet verified against the primary source for this page's date.
- The `mcp__<server>__<tool>` and `mcp__<server>__*` rule forms: [Claude Code docs: Configure permissions](https://code.claude.com/docs/en/permissions.md), verified 2026-11-09.
- The server names, URL and configuration values are illustrations; `example-db-mcp-server` is not a real package.

{% include page_nav.html %}
