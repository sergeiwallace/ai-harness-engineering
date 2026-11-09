---
title: "Chat, Projects, Cowork, Code and mobile"
parent: "Desktop, Web & Mobile Apps"
nav_order: 3
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Chat, Projects, Cowork, Code and mobile

The Claude apps on desktop, web and mobile now have several places to work, and each supports a different share of a harness.

- The surfaces: Chat, Projects, Cowork and the Code tab, plus Claude Code on the web and ways to reach a running session from a phone.
- For each one: what it is, what is new, and how the harness concepts from this site map onto it.
- At the end: Remote Control, which links a Claude Code session on your machine to the web and mobile apps.

## Surfaces at a glance

| Surface | Where | Where the work runs | Harness support |
|---|---|---|---|
| Chat | Web, desktop, mobile | Anthropic's servers | Instructions, memory, connectors, interactive apps |
| Classic Projects | Web, desktop, mobile | Anthropic's servers | Shared instructions and files |
| New Projects (beta) | claude.ai/code, the Code tab, mobile | Cloud Claude Code threads, or your computer | Project instructions and memory, repositories, connectors |
| Cowork | Desktop, web, mobile, the Chrome side panel | An isolated environment on Anthropic's servers | Skills, plugins and connectors from your claude.ai settings |
| Code tab | Desktop | Your machine, the cloud, SSH or WSL | Almost the whole harness |
| Claude Code on the web | Web | A cloud session | The repository's harness |
| Mobile | iOS and Android apps | The surface you connect to | Whatever that surface supports |

## If you reach Claude through an API key or a cloud provider

Much of this page needs a claude.ai subscription. Anthropic's feature-availability page lists these as "not reachable with an Anthropic Console API key or from a third-party provider":

- Cloud sessions and Claude Code on mobile.
- Claude Code Desktop, cloud scheduled tasks and Remote Control.
- Code Review, Ultrareview, the Chrome extension, computer use, Artifacts, voice dictation and Claude Code in Slack.

| Route | What you get |
|---|---|
| Claude Code CLI on any provider, Bedrock included | Everything local: sub-agents, hooks, commands, skills, `CLAUDE.md`, plugins, MCP servers, sandboxing and managed settings "work on every provider" |
| Claude Desktop on third-party providers (Bedrock, Google Cloud's Agent Platform, Microsoft Foundry, a gateway, or the Claude API with a Console key) | "The full Claude Desktop experience (Chat, Cowork, and Code, including file creation, multi-step research, and sub-agent coordination)", with a local device identity, conversations stored on your disk and configuration through device management or the admin console |
| Not available on those routes | Remote Control, cloud sessions, Dispatch and mobile Code; claude.ai account features such as Projects and memory are not listed for this mode (my reading of the sources) |

Anthropic also notes that standard Desktop with Cowork on Team or Enterprise "is simpler to deploy and releases new features more quickly than Claude Desktop on 3P."

## Chat

### What it is

- An ordinary conversation in the web, desktop or mobile app.
- On desktop, web and mobile, chat and Cowork now "share one home".

### What is new

- Chat and Cowork share one home on desktop, web and mobile, per the Cowork help article; the release notes date the rollout to Pro and Max from 2026-09-16 (wording not re-read). Code stays a separate tab.
- Memory, on by default for Free, Pro, Max and Ultra, saved as editable topics; each project has its own memory, and sensitive topics are off by default.
- Design, deck and document artifacts that can be created in any conversation (release notes, 2026-09-16; wording not re-read).
- Interactive connectors, built on [MCP Apps]({{ '/sections/19-chat-and-desktop-clients/mcp-apps.html' | relative_url }}), on web, desktop and mobile.

### How the harness maps onto it

| Concept | In Chat |
|---|---|
| Context files | Your preferences and memory act as standing context; keep them short |
| Skills | Skills from your claude.ai settings, read as procedures |
| Enforcement | None by program; you run the checklist |
| Connectors and MCP | Connectors from the directory, and custom connectors |
| Delegation | None you control |
| Human gates | You decide what to send or act on |

## Projects

Two versions of Projects exist side by side:

- Classic Projects, in Chat and, for now, in Cowork.
- New Projects, a public beta built on Claude Code threads.

### Classic Projects

Self-contained workspaces with their own chat histories, uploaded files (the knowledge base) and project instructions.

| Property | Detail |
|---|---|
| Plans | All plans; Free is capped at five projects |
| Large knowledge bases | On paid plans, when knowledge nears the context limit, Claude switches to a retrieval mode with "up to 10x" capacity |
| Sharing | View or edit sharing, with individuals or the whole organisation, on Team and Enterprise only |
| Cowork | Existing Cowork projects stay on this version for now; the new version "is expected to reach Cowork later" |

### New Projects (public beta)

A project is one ongoing conversation. Claude starts a thread for each task, usually a cloud Claude Code session, and threads run in parallel and keep going after you close the laptop.

- Availability: "public beta on Pro, Max, and Ultra plans and rolling out gradually", with a waitlist; "Projects aren't offered on Team or Enterprise plans yet."
- Where: claude.ai/code, the desktop app's Code tab, and the iOS and Android apps.
- Standing context:
  - Project instructions, up to 16,000 characters, sent to every new thread.
  - Project memory: files with a `MEMORY.md` index that every cloud thread reads at the start, separate from local Claude Code memory.
  - Repositories, uploaded files and Google Drive folders, and a Library tab for outputs.
- New projects default every thread to Opus at high effort.

#### What a cloud thread does and does not load

| Loads | Does not load |
|---|---|
| Each repository's `CLAUDE.md` | The repository's skills, `.claude/settings.json` and `.mcp.json` |
| claude.ai connectors | Anything from your local Claude Code setup |
| Project instructions and project memory | Permission rules: cloud threads run in auto mode, with approvals inside each thread |

Anthropic's documentation states the systems point directly: preferences Claude saves to memory "are instructions Claude keeps to, not enforced settings, so a thread limit you give this way isn't a hard cap."

#### Running a thread on your own computer

A project thread can run on your machine through Remote Control. That thread uses the machine's files, tools, MCP servers and Claude Code settings, "including its hooks and permission rules", and starts with the project instructions but not the project memory.

1. In the desktop app, turn on Settings, Code, "Use this computer from your phone and claude.ai".
2. Or, in a terminal, run `claude remote-control --spawn worktree`.

### How the harness maps onto Projects

| Concept | Classic Projects | New Projects, cloud thread | New Projects, thread on your computer |
|---|---|---|---|
| Context files | Project instructions | Project instructions, project memory, `CLAUDE.md` | Project instructions, `CLAUDE.md` |
| Skills | Your claude.ai skills | Not the repository's skills | Your local skills |
| Enforcement | None by program | No local hooks or permission rules | Your local hooks and permission rules |
| Connectors and MCP | claude.ai connectors | claude.ai connectors | Your local MCP servers |
| Delegation | None you control | One thread per task, in parallel | One thread per task |

## Cowork

### What it is

In Anthropic's words, "Claude Cowork uses the same agentic architecture that powers Claude Code, with no terminal required": Claude plans and runs multi-step knowledge-work tasks and delivers finished files.

| Property | Detail |
|---|---|
| Surfaces | Desktop app (macOS, Windows; Linux in beta), web, mobile, and the Claude in Chrome side panel |
| Where it runs | "On Anthropic's servers, in an isolated environment", and it continues if you close the laptop |
| Local files | Desktop only: "Claude can only read and write files in folders you've connected" |
| Sub-agents | "Claude may coordinate multiple sub-agents working simultaneously" |
| Scheduled tasks | `/schedule`, or Scheduled in the sidebar; "Scheduled tasks run in the cloud, with no device online" |
| Plans | Paid plans only (Pro, Max, Ultra, Team, Enterprise); web and mobile access depend on plan and admin settings |

### What is new

- Cowork on mobile and web, with sessions running remotely, in beta (release notes, 2026-07-07; wording not re-read).
- Cowork features available from any conversation, rolling out to Pro and Max on web, desktop and mobile (release notes, 2026-09-16; wording not re-read).
- Interactive connectors, per Anthropic's January 2026 announcement.

### Approvals and safety

| Approval mode | Behaviour |
|---|---|
| Manual | Asks each time |
| Auto | Keeps working, reviews each action and blocks unsafe ones; Anthropic adds that "no mode replaces your judgment" |
| Skip | Only when you trust everything involved |

- Team and Enterprise admins control whether "Automatically approve" is available.
- Cowork asks for explicit permission before permanently deleting files.
- "Isolation protects your computer; it doesn't change what Claude can read or do through the access you've granted."
- Network egress settings do not apply to web fetch, web search, MCP servers or Chrome.

### How the harness maps onto it

| Concept | In Cowork |
|---|---|
| Context files | Memory shared with chat; your claude.ai settings |
| Skills and plugins | Plugins bundle skills, connectors and sub-agents; Cowork takes them from your claude.ai account's Customize settings, not from `~/.claude` |
| Enforcement | The approval modes and the isolated environment; not your local Claude Code hooks |
| Connectors and MCP | The "+" menu or Customize, Connectors |
| Delegation | Sub-agents that Cowork coordinates |
| Human gates | Manual approval mode, and deletion prompts |

## Code tab and Claude Code on the web

### The Code tab

"In the Claude Desktop app, Claude Code is the Code tab." It runs the same engine as the terminal, so the whole harness applies.

| Feature | Detail |
|---|---|
| Sessions | Each conversation is a session with its own folder; several can run in parallel |
| Worktrees | Turn on Worktree to isolate each session in a git worktree, by default under `.claude/worktrees/` in the project |
| Environments | Local, Cloud (Anthropic-managed, continues when the app is closed), SSH, WSL |
| Review and shipping | A diff view with line comments, a Preview pane that runs your dev server and checks after edits, and pull request monitoring with auto-fix or merge when ready (needs `gh`) |
| Skills and plugins | Local sessions load personal skills from `~/.claude/skills/`; plugins add skills, agents, hooks, MCP servers and language-server configuration |
| Permissions | Bypass permissions can be self-enabled on Pro, Max and Ultra; on Team and Enterprise, organisation policy controls it |

### Claude Code on the web

Claude Code sessions that run in the cloud.

- A cloud session runs in an isolated, Anthropic-managed virtual machine, or in your organisation's own environment, and keeps running after you close the laptop.
- The repository's `CLAUDE.md`, its `.claude/agents/` sub-agents and its `.claude/settings.json` apply, for single-repository sessions outside a project; `SessionStart` hooks run, with a timeout.
- Local-only plugins and commands such as `/plugin` and `/resume` do not.
- Start one from claude.ai/code, the mobile app, the desktop app's Cloud environment, or `claude --cloud "task"`; pull one back to your terminal with `/teleport`.
- It needs a claude.ai sign-in, and organisations with Zero Data Retention, HIPAA or customer-managed keys cannot use cloud sessions.
- Plugins from your local settings do not load in a cloud session.
- Team and Enterprise organisations can run cloud sessions in their own environment ("self-hosted environments").

## Mobile

| What you can do from the phone | Where the work runs |
|---|---|
| Chat, Projects and interactive connectors | Anthropic's servers |
| Cowork | An isolated environment on Anthropic's servers |
| New Projects threads | The cloud, or your computer through Remote Control |
| Dispatch: message a task that runs on your desktop and can start Code sessions (Pro, Max and Ultra; "not available on Team or Enterprise plans") | Your machine, through the desktop app |
| Remote Control: drive a running Claude Code session | Your machine |

Remote Control and Dispatch send push notifications to the phone when a long task finishes or Claude needs a decision; permission prompts and questions stay open until you answer from any device.

## Remote Control

### What it is

- Remote Control connects claude.ai/code or the Claude iOS and Android apps to a Claude Code session running on your machine.
- Claude keeps running locally, so your files, MCP servers, tools and project configuration stay on your machine.
- The conversation, and the progress of sub-agents, stay in sync across the terminal, browser and phone.

### Starting it

```text
claude remote-control          # server mode; can serve several sessions
claude --remote-control        # or --rc: an interactive session that is also reachable remotely
/remote-control                # or /rc: inside a session already running
```

The Code tab in the desktop app and the VS Code extension also accept `/remote-control` and `/rc`. In server mode, `--spawn worktree` gives each session started on demand its own git worktree.

### Availability

| Situation | Remote Control |
|---|---|
| Pro, Max or Ultra subscription, signed in with claude.ai | Available |
| Team or Enterprise | Available once an Owner turns on the Remote Control toggle in Claude Code admin settings; custom roles must grant both the cloud sessions permission and the "Claude Code remote control" permission |
| Signed in with an API key (`ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` or an `apiKeyHelper`) | Not available: "Remote Control requires claude.ai subscription auth." |
| Amazon Bedrock, Google Cloud Vertex AI, Microsoft Foundry | Not available |
| `ANTHROPIC_BASE_URL` pointing at an LLM gateway or proxy, or an enterprise Claude apps gateway | Not available |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` or `DISABLE_GROWTHBOOK` set | Not available |
| Organisations with Zero Data Retention or the HIPAA configuration | Cannot enable it |
| The `disableRemoteControl` setting | Turns it off |

So on a company setup that reaches the model through an API key, a cloud provider or a gateway, Remote Control does not work; on a personal subscription it does, and on Team or Enterprise it depends on the admin settings.

### Security settings

- Trusted Devices is off by default. An Owner can require a verified device before anyone views or steers a Remote Control session from claude.ai, mobile or the desktop app (Organization settings, Capabilities, Remote sessions).
- Code and files stay on your machine; the remote surface sends messages and attachments to the local session, which downloads attached files as `@` references.

### Ways to work away from the terminal

| Mode | Trigger | Runs on |
|---|---|---|
| Dispatch | Message a task from the Claude mobile app | Your machine, through the desktop app |
| Remote Control | Drive a running session from claude.ai/code or mobile | Your machine (CLI, desktop app, VS Code) |
| Channels | Push events from Telegram, Discord or your server | Your machine (CLI) |
| Slack | Mention @Claude in a channel | Anthropic's cloud |
| Self-hosted environments | Start a cloud session in your organisation's environment | Your organisation's infrastructure (Team, Enterprise) |
| Scheduled tasks | A schedule | CLI, desktop app, or the cloud |

## How the harness maps across surfaces

| Concept | Chat and classic Projects | New Projects, cloud thread | Cowork | Code tab, local | Remote Control from mobile |
|---|---|---|---|---|---|
| Context files | Preferences, memory, project instructions | Project instructions and memory, `CLAUDE.md` | Memory, claude.ai settings | `CLAUDE.md` | Your local `CLAUDE.md` |
| Skills | claude.ai skills | Not the repository's skills | claude.ai skills and plugins | `~/.claude/skills/` and plugins | Your local skills |
| Enforcement (hooks, linters, tests) | None by program | No local hooks; CI still runs | Approval modes and isolation | Full | Your local hooks |
| Connectors and MCP | claude.ai connectors | claude.ai connectors | Connectors | Local and remote MCP servers | Your local MCP servers |
| Delegation | None you control | A thread per task | Sub-agents | Sub-agents and parallel sessions | Sub-agents, in sync across devices |
| Human gates | Your review | Approvals inside each thread | Manual mode, deletion prompts | Permission rules | Prompts answered from any device |

<!-- IMAGE NEEDED: a screenshot of the Claude desktop app showing the Chat, Cowork and Code tabs, or the Remote Control session view on a phone. -->

## References

- [Claude Code docs: Continue local sessions from any device with Remote Control](https://code.claude.com/docs/en/remote-control), read 2026-11-09; Remote Control commands, availability, settings and the table of ways to work away from the terminal.
- [Claude Code docs: Let Claude coordinate ongoing work with Projects](https://code.claude.com/docs/en/claude-projects), read 2026-11-09; new Projects.
- [Claude Code docs: Claude Code Desktop](https://code.claude.com/docs/en/desktop), read 2026-11-09; the Code tab, Dispatch, and where Cowork takes its skills from.
- [Claude Code docs: Feature availability](https://code.claude.com/docs/en/feature-availability), read 2026-11-09; what needs a claude.ai subscription.
- [Claude Code docs: Use Claude Code in the cloud](https://code.claude.com/docs/en/claude-code-on-the-web), read 2026-11-09.
- [Claude Code docs: Claude Code on mobile](https://code.claude.com/docs/en/mobile), read 2026-11-09.
- [Claude Desktop on third-party providers: Overview](https://claude.com/docs/third-party/claude-desktop/overview), read 2026-11-09.
- [Claude help: What are projects?](https://support.claude.com/en/articles/9517075-what-are-projects), read 2026-11-09 through a summary; classic Projects.
- [Claude help: Get started with Cowork](https://support.claude.com/en/articles/13345190-get-started-with-cowork), read 2026-11-09 through a summary; Cowork.
- [Claude help: Release notes](https://support.claude.com/en/articles/12138966-release-notes), read through a summary on 2026-11-09; dated entries are marked where the wording was not re-read.
- [Claude Code docs: Plugins](https://code.claude.com/docs/en/plugins/overview.md), verified 2026-11-08.
- [Anthropic, Your favorite work tools are now interactive connectors inside Claude](https://claude.com/resources/articles/interactive-tools-in-claude), 2026-01-26.

{% include page_nav.html %}
