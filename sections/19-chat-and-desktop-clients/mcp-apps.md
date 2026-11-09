---
title: "MCP Apps: interactive UI in chat clients"
parent: "Desktop, Web & Mobile Apps"
nav_order: 2
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# MCP Apps: interactive UI in chat clients

Related: [Claude Code mods and function hooks]({{ '/sections/05-skills-and-plugins/mods-and-function-hooks.html' | relative_url }}).

An MCP App is an MCP server tool that returns an interactive interface along with its result: a form, a chart, a map, a document viewer, rendered in a sandboxed frame inside the conversation. It is the chat clients' way to extend their own interface, much as a mod extends the Claude Code terminal interface.

This page covers:

1. What an MCP App is, and where it runs.
2. How it works.
3. How to build and test one.
4. Its security model.
5. How it compares with Claude Code mods.

<!-- IMAGE NEEDED: a screenshot of an MCP App rendered inside a Claude conversation, for example an interactive chart or a form returned by a connector, with the surrounding chat visible. -->

## What an MCP App is

MCP Apps is an official extension to the Model Context Protocol.

- Its stable specification is version 2026-01-26, discussed as SEP-1865.
- It combines two existing MCP primitives, a tool and a resource, with a new rendering step in the host.

| Part | What it is | Where it lives |
|---|---|---|
| Tool | An ordinary MCP tool whose description carries `_meta.ui.resourceUri` | The MCP server |
| UI resource | An HTML page served under a `ui://` URI, usually bundled with its scripts and styles | The MCP server |
| View | The running UI, inside a sandboxed iframe | The host's window |
| Host | The chat client that fetches, renders and brokers messages for the View | Claude, ChatGPT, VS Code and others |

The View talks to the host over `postMessage`, using JSON-RPC in a dialect of MCP: some methods are shared with core MCP (`tools/call`), and most are new, with a `ui/` prefix.

## Where it runs

### Claude

| Surface | Interactive connectors | Notes |
|---|---|---|
| Claude web (claude.ai) | Yes | Featured apps in the connector directory |
| Claude desktop app | Yes | |
| Claude mobile apps | Yes | |
| Cowork | Yes | Anthropic's January 2026 announcement says interactive connectors are "now available on Claude Cowork" |

Anthropic's announcement lists the plans as "Free, Pro, Max, Ultra, Team, and Enterprise", with launch partners including Amplitude, Asana, Box, Canva, Clay, Figma, Hex, monday.com and Slack. Users enable them from the directory at claude.ai, in the featured section.

Your own MCP App reaches Claude as a custom connector.

- The MCP build guide says custom connectors are available on paid plans (Pro, Max or Team).
- I have not checked this against Anthropic's current help pages.

### Other hosts

The MCP Apps documentation and the ext-apps repository list these hosts, among others:

- Claude and Claude Desktop.
- ChatGPT.
- VS Code with GitHub Copilot, and Microsoft 365 Copilot.
- Goose, Postman, MCPJam and Archestra.AI.

Support varies by host. The MCP site keeps a client matrix of which clients support which extensions.

## How it works

1. The tool declares its UI.
   - Its description includes `_meta.ui.resourceUri`, pointing at a `ui://` resource.
   - The host can preload the resource before the tool is called, which lets it stream tool input into the View.
2. The host fetches the UI resource from the server.
   - The resource is HTML, usually with its scripts and styles bundled in.
   - The resource's `_meta.ui` can declare `csp` (which external origins it may load from) and `permissions` (for example the microphone or camera).
3. The host renders the View in a sandbox.
   - Web hosts must wrap the View in a sandbox proxy on a different origin from the host. Desktop hosts can render it in an iframe directly.
4. The View and the host complete a handshake.
   - The View sends `ui/initialize`, the host replies with its context (such as the theme), and the View sends `ui/notifications/initialized`. The host may send nothing to the View before that.
5. The host pushes the tool's data.
   - It sends `ui/notifications/tool-input` with the arguments, then `ui/notifications/tool-result` with the result.
6. The user works in the View, and the View talks back.
   - It can call tools on its own server with `tools/call`, which the host forwards.
   - It can add a message to the chat (`ui/message`), update what the model sees on later turns (`ui/update-model-context`), ask the host to open a link (`ui/open-link`), or ask for inline, fullscreen or picture-in-picture display (`ui/request-display-mode`).
7. The host tears the View down.
   - It sends `ui/resource-teardown` and should wait for the View's response.

### The main messages

| Message | Direction | Purpose |
|---|---|---|
| `ui/initialize` | View to host | Handshake; the host returns its context |
| `ui/notifications/tool-input` | Host to View | The tool's arguments, sent once after initialization |
| `ui/notifications/tool-input-partial` | Host to View | Optional, best-effort streaming of partial arguments |
| `ui/notifications/tool-result` | Host to View | The tool's result |
| `tools/call` | View to host | Call a tool on the same server |
| `ui/message` | View to host | Add content to the conversation; the host may ask the user first |
| `ui/update-model-context` | View to host | Context for later turns; each call replaces the previous one |
| `ui/open-link` | View to host | Ask the host to open a URL |
| `ui/notifications/size-changed` | View to host | Report a new content size so the host can resize the frame |
| `ui/resource-teardown` | Host to View | The View is about to be removed |

## Building one

The build guide's example is a server that returns the current time, with a View that shows it and a button to refresh it. The project has three parts:

- `server.ts`: registers the tool and serves the UI resource.
- `mcp-app.html` and `src/mcp-app.ts`: the View.
- A Vite build that bundles the View into one HTML file under `dist/`.

### Install

```bash
npm install @modelcontextprotocol/ext-apps @modelcontextprotocol/sdk
npm install -D typescript vite vite-plugin-singlefile express cors @types/express @types/cors tsx
```

### The server

The two helpers from `@modelcontextprotocol/ext-apps/server` register the tool with its UI reference and serve the HTML. Shortened from the guide:

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import {
  registerAppTool,
  registerAppResource,
  RESOURCE_MIME_TYPE,
} from "@modelcontextprotocol/ext-apps/server";
import fs from "node:fs/promises";
import path from "node:path";

const server = new McpServer({ name: "My MCP App Server", version: "1.0.0" });
const resourceUri = "ui://get-time/mcp-app.html";

registerAppTool(
  server,
  "get-time",
  {
    title: "Get Time",
    description: "Returns the current server time.",
    inputSchema: {},
    _meta: { ui: { resourceUri } },
  },
  async () => ({ content: [{ type: "text", text: new Date().toISOString() }] }),
);

registerAppResource(server, resourceUri, resourceUri, { mimeType: RESOURCE_MIME_TYPE }, async () => ({
  contents: [{
    uri: resourceUri,
    mimeType: RESOURCE_MIME_TYPE,
    text: await fs.readFile(path.join(import.meta.dirname, "dist", "mcp-app.html"), "utf-8"),
  }],
}));

// The guide then exposes the server over Streamable HTTP with Express, on http://localhost:3001/mcp.
```

### The View

```typescript
import { App } from "@modelcontextprotocol/ext-apps";

const app = new App({ name: "Get Time App", version: "1.0.0" });
app.connect();

// The host pushes the first tool result when the View renders.
app.ontoolresult = (result) => {
  document.getElementById("server-time")!.textContent =
    result.content?.find((c) => c.type === "text")?.text ?? "[ERROR]";
};

// The View can call the server's tools itself; each call is a round trip.
document.getElementById("get-time-btn")!.addEventListener("click", async () => {
  const result = await app.callServerTool({ name: "get-time", arguments: {} });
  document.getElementById("server-time")!.textContent =
    result.content?.find((c) => c.type === "text")?.text ?? "[ERROR]";
});
```

The guide bundles the HTML, script and styles into one file with `vite-plugin-singlefile`, because the default content security policy blocks external assets.

### Testing

1. In the ext-apps test host.
   - Clone the ext-apps repository, then from `examples/basic-host` run the command below and open port 8080 on localhost in a browser.
2. In Claude web or desktop.
   - Expose the local server with a tunnel, then add the tunnel's URL in Claude under Settings, Connectors, Add custom connector.

```bash
# 1. The test host, pointed at your server
SERVERS='["http://localhost:3001/mcp"]' npm start

# 2. A public tunnel to your server, for Claude
npx cloudflared tunnel --url http://localhost:3001
```

<details markdown="1">
<summary>Scaffolding one with Claude Code</summary>

The ext-apps repository ships a `create-mcp-app` skill. In Claude Code:

```text
/plugin marketplace add modelcontextprotocol/ext-apps
/plugin install mcp-apps@modelcontextprotocol-ext-apps
```

Then ask for an app, for example "Create an MCP App that displays a color picker", and the agent scaffolds the server, the View and the build configuration.

</details>

## Security and trust

### What the sandbox enforces

| Control | Rule in the specification |
|---|---|
| Isolation | The View runs in a sandboxed iframe and cannot reach the host page's DOM, cookies or local storage, or navigate the parent page |
| Separate origin | On the web, the sandbox proxy must have a different origin from the host |
| Network and assets | With no `csp` declared, the host must apply a restrictive default starting `default-src 'none'`, with `connect-src 'none'`; declared `connectDomains`, `resourceDomains`, `frameDomains` and `baseUriDomains` widen it, and hosts must not allow undeclared domains |
| Device access | Camera or microphone only through declared `permissions`, which the host may grant |
| Tool reach | A View can call tools on its own server only; calls across servers from a View are always blocked |
| Tool visibility | `_meta.ui.visibility` defaults to `["model", "app"]`; a tool without `"model"` is hidden from the model, and a tool without `"app"` cannot be called from the View |

### What it does not settle

- User approval of the View's own tool calls is left to the host.
  - The specification says hosts "MAY decide to block some messages or subject them to further user approval", and may ask the user before a `ui/message`.
- Prompt injection is not addressed.
  - The specification does not mention it. Text a View adds to the conversation, or places in model context, comes from the server, so treat it like any tool result from a third party.
- Misleading content is possible.
  - The threat model notes that a View can still show misleading content, such as an interface that imitates the host.

> Warning: whether a View's tool calls need your approval each time is up to the host, and the specification does not require it. Connect only servers you would trust with all of their tools, and check what your host asks you to approve.

## MCP Apps and Claude Code mods

Both let a third party extend a client's interface, so they play similar roles. They are different mechanisms with different trust models.

| | MCP App | Claude Code mod |
|---|---|---|
| Who provides it | A remote or local MCP server, often run by a vendor | A local plugin you install |
| Where it runs | A sandboxed iframe inside the conversation | Code loaded into the Claude Code client on your machine |
| What it can change | Its own panel in the conversation, plus messages and context through the host | The client's own behaviour and its terminal interface, such as panels beside the chat and hooks on tool calls |
| How it reaches data | Only its own server's tools, and origins its policy declares | Whatever the client and your machine allow it |
| Trust model | Deny-by-default sandbox; the host brokers every message | Trusted local code, like any plugin you install |
| Typical uses | Charts, forms, previews, approvals inside a chat | Status panels, custom views of the session, guards on the agent's actions |
| Where it works | Any host that supports the extension | Claude Code only |

The [mods page]({{ '/sections/05-skills-and-plugins/mods-and-function-hooks.html' | relative_url }}) covers how mods are built and loaded.

## Pitfalls

| Pitfall | What happens | Fix |
|---|---|---|
| External scripts or styles without a `csp` | The default policy blocks them, and the View renders blank or unstyled | Bundle into one HTML file, or declare `resourceDomains` and `connectDomains` |
| A local server added to Claude | Claude cannot reach `localhost` | Use a tunnel such as `cloudflared`, or test in the basic host first |
| A View-only tool left visible to the model | The model calls a tool meant only for the interface | Set `_meta.ui.visibility` to `["app"]` |
| Sending to the View too early | The spec forbids messages before `ui/notifications/initialized`, so they can be lost | Wait for the handshake; the `App` class handles it |
| Chatty `callServerTool` use | Every call is a round trip through the host to the server | Batch requests, and show loading state |
| Repeated `ui/update-model-context` | Each call replaces the previous context | Send the full context each time |
| Assuming every host behaves alike | Display modes, approvals and permissions vary by host | Test in each host you target |

## Related

- [Claude Code mods and function hooks]({{ '/sections/05-skills-and-plugins/mods-and-function-hooks.html' | relative_url }}): the Claude Code counterpart.
- [Desktop, Web & Mobile Apps]({{ '/sections/19-chat-and-desktop-clients/' | relative_url }}): which parts of a harness carry over to these clients.
- [Harness habits in a chat app]({{ '/sections/19-chat-and-desktop-clients/harness-habits-in-a-chat-app.html' | relative_url }}).

## References

- [MCP Apps overview, Model Context Protocol docs](https://modelcontextprotocol.io/docs/extensions/apps), read 2026-11-09.
- [Build an MCP App, Model Context Protocol docs](https://modelcontextprotocol.io/extensions/apps/build), read 2026-11-09; source of the code and test steps above.
- [MCP Apps specification, version 2026-01-26](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx), read 2026-11-09; source of the sandbox, policy, visibility and message rules.
- [The ext-apps repository](https://github.com/modelcontextprotocol/ext-apps), read 2026-11-09; SDK packages and supported hosts.
- [Anthropic, Your favorite work tools are now interactive connectors inside Claude](https://claude.com/resources/articles/interactive-tools-in-claude), 2026-01-26, read 2026-11-09; surfaces, plans and launch partners.
- [Anthropic help: Getting started with custom connectors using remote MCP](https://support.anthropic.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp), linked from the build guide; not re-checked for this page.

{% include page_nav.html %}
