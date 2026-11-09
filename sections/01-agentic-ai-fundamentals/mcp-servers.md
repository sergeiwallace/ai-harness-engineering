---
title: "MCP servers"
parent: "Tools and tool calling"
grand_parent: "Agentic AI fundamentals"
nav_order: 1
---

{% include page_nav.html %}

# MCP servers

As of 2026-11-09.

The Model Context Protocol (MCP) is an open standard for plugging an outside service into an AI client as a set of tools. One server can serve Claude Code, Codex, a desktop chat app and an IDE alike, so a tool written once works everywhere. This page covers how MCP is put together, how to add a server, what a server costs, and when a server is the wrong answer. It is part of [Tools and tool calling]({{ '/sections/01-agentic-ai-fundamentals/tools.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The shape of MCP

### Hosts, clients and servers

![Sequence diagram: at startup the host client starts and initializes the MCP server, asks it for its tools, receives names, descriptions and schemas, and puts the tool definitions in the model's context; later the model writes a tool_use for run_query, the host sends tools/call over JSON-RPC after its permission checks and hooks, the server runs a SELECT as a read-only database user, and the rows flow back as the result and then as a tool_result to the model]({{ '/assets/images/01-agentic-ai-fundamentals/mcp-startup-and-call.svg' | relative_url }}){: style="width: 100%" }
*Figure: one MCP server's life in a session, from registration to a single call. The red arrow is where the server's own limits apply.*

1. At startup the host starts each configured server and asks it for its tools.
2. The server answers with each tool's name, description and input schema.
3. The host adds those definitions to the model's context, where they stay on every turn.
4. When the model calls one of the tools, the host runs its permission checks and hooks, then forwards the call to the server.
5. The server does the work against its service and returns the result, which the host passes to the model as a `tool_result`.

| Role | What it is | Example |
|---|---|---|
| Host | the application the person uses | Claude Code |
| Client | the connector inside the host, one per server | started by the host from its config |
| Server | a program that exposes capabilities | a server wrapping a database |

Messages between client and server use JSON-RPC 2.0. The model never talks to a server directly: the server's tools appear in the model's context like built-in ones, and when the model calls one, the host's client forwards the call.

### Transports

#### stdio

The host starts the server as a child process and talks to it over standard input and output. Simple, local, no network port. Most servers used by one developer run this way.

#### HTTP

The server runs as a web service and the client connects over HTTP, with streaming for long responses. This suits shared or hosted servers, and needs authentication, often OAuth.

### What a server can offer

| Primitive | Controlled by | What it is |
|---|---|---|
| Tools | the model | actions the model can call, with a schema, exactly like built-in tools |
| Resources | the application or person | data the host can attach to the context, such as a file or a record |
| Prompts | the person | reusable prompt templates, often shown as slash commands |

Most servers in coding use offer tools only.

## Adding a server

### From the command line

Claude Code's `claude mcp add` registers a server. A local stdio server takes the command that starts it; a remote one takes its URL.

```bash
# a local server, started as a child process
claude mcp add my-db -- npx -y example-db-mcp-server --readonly

# a remote server over HTTP
claude mcp add --transport http tracker https://mcp.example.com/mcp

# see what is registered and whether each server connected
claude mcp list
```

### In a project file

A project can check its servers into the repository so everyone who opens it gets the same tools. Claude Code reads a `.mcp.json` file at the project root:

```json
{
  "mcpServers": {
    "my-db": {
      "command": "npx",
      "args": ["-y", "example-db-mcp-server", "--readonly"],
      "env": { "DB_URL": "${DB_URL}" }
    }
  }
}
```

The `${DB_URL}` reference reads the value from the environment, so the secret never lands in the file.

### Scopes

| Scope | Where it is stored | Who gets it |
|---|---|---|
| Local | your user config, for this project only | you, in this project |
| Project | `.mcp.json` in the repository | everyone who clones it, after they approve it |
| User | your user config, for every project | you, everywhere |

Claude Code asks before it trusts a project-scoped server for the first time, because a cloned repository could otherwise start any program.

### Worked example: a read-only database tool

A team wants the agent to answer "how many orders failed yesterday" without anyone pasting query results. They add a database MCP server in read-only mode at project scope. Now the agent can list tables, read a schema and run a `SELECT`. Two choices made it safe: the `--readonly` flag, enforced by the server rather than by an instruction, and a database user whose grants allow only reads. If the model is ever talked into writing, the server and the database both refuse.

## What a server costs

### Context

Every tool a server exposes adds its definition to the model's context. A server with forty tools costs forty definitions on every turn until it is disconnected. Some clients now defer loading tool definitions until they are needed, which reduces the cost; [Tools and tool calling]({{ '/sections/01-agentic-ai-fundamentals/tools.html' | relative_url }}#what-tools-cost) has an estimate.

### Trust

A server is a program running with your permissions, or a remote service receiving your data. Its tool descriptions and results go into the model's context, where text can steer the model. A malicious or compromised server can therefore both act and inject instructions.

> Treat an MCP server like a dependency you install: know who wrote it, pin its version, and give it the narrowest credentials that work.
{: style="border-left: 4px solid #b91c1c; background: #fef2f2; padding: 0.75rem 1rem;"}

### Reliability

A local server that fails to start, or a remote one that times out, removes its tools mid-session. `claude mcp list` and the `/mcp` command show which servers are connected.

## When not to use a server

<details markdown="1">
<summary>A command-line tool the shell can run is often enough</summary>

If a service already has a good command-line tool, the agent can call it through the shell with no server at all. The command line costs nothing in context until it is used, its output can be trimmed with ordinary flags, and its permissions are the ones you already manage. A server earns its place when it gives a structured interface the command line lacks, when it holds a session or credentials the shell should not see, or when the same tools must work in clients that have no shell, such as a desktop chat app.
</details>

| Need | Better fit |
|---|---|
| Call a service that has a solid CLI | the shell, optionally wrapped in a skill |
| A procedure of several steps | a skill |
| The same tool in a chat app, an IDE and a terminal agent | an MCP server |
| Structured access with credentials the agent should not hold | an MCP server |

## Sources

- The Model Context Protocol specification: <https://modelcontextprotocol.io/specification>, not yet verified against the primary source for this page's date.
- Claude Code MCP configuration, scopes and `claude mcp` commands: <https://code.claude.com/docs/en/mcp.md>, not yet verified against the primary source for this page's date.
- The server names, URL and configuration values are illustrations; `example-db-mcp-server` is not a real package.

{% include page_nav.html %}
