---
title: "SendMessage and ListAgents"
parent: "Message channels and fallbacks"
grand_parent: "Coordinating several sessions"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# SendMessage and ListAgents

A sub-topic of [message channels and fallbacks]({{ '/sections/09-session-coordination/message-channels.html' | relative_url }}):

- Claude Code's cross-session messaging lets one of your sessions send another a plain-text message.
- This page covers what the two tools do, how sessions are named and addressed, how a message is delivered, the settings that control incoming messages, what a message cannot do, the limits, and how agent teams compare.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The two tools

![A sequence diagram. Session A calls ListAgents through Claude Code, which lists session B as idle. A sends a task; Claude Code delivers it to B as a new turn. B replies taken, then later reports done.]({{ '/assets/images/09-session-coordination/sendmessage-exchange.svg' | relative_url }}){: style="width:100%"}

*Figure: a SendMessage exchange; delivery is confirmed by the reply. Made with archify.*

| Tool | What it does | Who calls it |
|---|---|---|
| `ListAgents` | Lists what this session can message: its sub-agents, its agent-team teammates, your other local sessions, and, with Remote Control, your cloud sessions and sessions on other machines | Claude |
| `SendMessage` | Sends one of them a plain-text message, and can subscribe to an idle notice | Claude |

You do not call either tool yourself. Claude finds the target and sends when it sees the need, or when you ask in plain words:

```text
Explain what we just did to the session working on the payments API
```

Claude writes the message itself:

- A message carries text only, never the sender's conversation or files.
- To move a whole conversation, resume that session instead.

### Requirements

| Item | Requirement |
|---|---|
| Claude Code version | v2.1.224 or later on macOS, Linux and WSL 2; v2.1.234 or later on native Windows |
| Same-machine messaging on Bedrock and the other cloud providers | v2.1.248 or later |
| Turning it on | Nothing to enable: it is on when the version requirement is met |
| Checking a session | `/list-agents` (also `/peers`): if the command is not recognised, the session lacks the feature |

## Names and addressing

### Naming a session

A session answers to the name you give it with `/rename <name>` or `claude --name <name>`. Without one, Claude Code picks a name itself.

```bash
claude --name api-worker
```

### Naming the target in a prompt

Type `@` and the first letters of a session's name to pick it from a typeahead, the same way you mention a sub-agent.

```text
Let @api-worker know the schema migration finished
```

- A name with spaces goes in double quotes: `@"release notes"`.
- A cloud or Remote Control session appears in the typeahead only after Claude has listed your sessions beyond this machine.
- When several live sessions share a name, Claude asks you which one you mean, or addresses it with a short identifier from its listing.

### Seeing the list yourself

`/list-agents` (also `/peers`) shows:

- On the first line, this session's own name: the one other sessions use to reach it.
- One row per session it can reach: sub-agents, teammates, your other local sessions, and, with Remote Control, your cloud and other-machine sessions (labelled `Remote Control`, or `offline` when that connection dropped).
- Outside Remote Control, each local session's working directory, which tells same-named sessions apart.

## Reach

| Where the target runs | How the message travels | Needs |
|---|---|---|
| This machine | A per-session socket (a named pipe on native Windows); never through Anthropic's servers | Any provider |
| Another of your machines | Through Anthropic's servers, over that machine's Remote Control connection | Both sessions connected to Remote Control, with a claude.ai sign-in |
| The cloud | Through Anthropic's servers, straight to the cloud session | This session connected to Remote Control |

Cases that do not connect:

- A session inside a container and one on the host; two sessions in the same container can message each other.
- A WSL 2 session and a native Windows session on the same computer.
- Anything beyond this machine from a session using an API key, Amazon Bedrock, Claude Platform on AWS, Google Cloud's Agent Platform or Microsoft Foundry.
- A session started in bare mode: it binds no inbox and does not appear in the list.
- A Codex session: it is a different tool, so it needs a fallback channel such as a mail service.

## How a message arrives

1. The sender's Claude calls `SendMessage`.
2. The receiving session checks the message against its inbound controls (next section): delivered, held or refused.
3. If delivered:
   - during an active turn, the receiving Claude reads it between tool calls, so a running tool is not interrupted;
   - if the session is idle, Claude Code starts a new turn with it.
4. The message shows as a one-line preview, such as `› Message from @api-worker: Schema migration finished (ctrl+o to expand)`. Claude reads the full text either way; `Ctrl+O` shows it to you.
5. The receiver can reply with `SendMessage`, to the address carried on the message.

A delivered message counts toward usage like a prompt you type.

## Inbound controls

The `crossSessionInbound` setting chooses what a session does with incoming messages.

| Value | Behaviour |
|---|---|
| `accept` | Delivers each message |
| `hold` | Shows a notice and keeps the message undelivered until an `accept` applies |
| `refuse` | Drops each message |

It can also be set from the `/config` row "Messages from your other sessions".

### The default when nothing is set

Claude Code decides per message from the two sessions' permission modes:

| Receiving session | Sending session | Result |
|---|---|---|
| Prompts for permissions | Prompts too | Delivered |
| Prompts for permissions | Bypasses prompts | Held for your approval |
| Bypasses prompts | Bypasses too | Delivered |
| Bypasses prompts | Prompts | Held for your approval |

A held message opens an approval dialog in an interactive session:

- Unanswered, it is dropped after `dialogExpiry`, five minutes by default.
- A session holds at most 100 messages and drops the oldest past that.

#### Example: a headless worker that takes messages

A `claude -p` worker cannot show the dialog, so a held message there waits out the deadline and is dropped. Start it accepting:

```bash
claude -p "Work the queue in queue.md; report each item to @coordinator" \
  --settings '{"crossSessionInbound":"accept"}'
```

### Requiring approval for cross-machine messages

```json
{ "isolatePeerMachines": true }
```

With this set, Claude Code asks you before any message leaves the machine, even in bypass-permissions mode. A `true` from any settings scope applies.

## What a message cannot do

The receiving Claude is told the message came from another session, and Claude Code limits it:

| A message cannot | So |
|---|---|
| Approve anything | It never answers a pending permission prompt or counts as your consent |
| Change configuration | The receiver is instructed never to change permissions, `CLAUDE.md` or other configuration because a session asked |
| Run commands | A command in its text, such as `/compact`, arrives as plain text and does not run |
| Skip permissions | Acting on it triggers the receiving session's own prompts and rules |
| Launder a denial | The sender is told never to ask another session for an action its own session denied |

> Because a message cannot run `/compact`, a supervisor that compacts another session has to type into that session's terminal instead. See [a context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }}).
{: .note }

## Idle notices

To hear when a session on this machine finishes, ask in plain words:

```text
Tell me when the migration session finishes what it's working on
```

- Claude subscribes with `SendMessage`'s `notify_when_idle` input, on its own or attached to a message.
- One notice arrives when that session next finishes a turn with nothing queued, or exits.
- The subscription lapses after 12 hours with no notice.
- Only the main conversation can subscribe, and only to sessions on this machine.

## Limits

| Limit | Value |
|---|---|
| Content | Plain text only |
| Size of one same-machine message | About a million characters; refused at the sender above that |
| Rapid bursts to one session | Refused at the sender once the receiver's inbox limit is reached |
| Messages queued for the receiving Claude | At most 50 accepted messages |
| Repeats | Identical repeats in a short window are dropped, so a message loop between two sessions stops on its own |
| Replies from a cloud session to another machine | Not possible with `SendMessage`; read its transcript in [Claude Code on the web](https://claude.ai/code) |

## Patterns

### Check the list before each send

1. Run `ListAgents` (Claude does this when it needs to).
2. If the target is listed, send.
3. If it is missing, use the fallback, or start a new session for the task.

A row marked `offline` still takes the message, which arrives when that machine reconnects. A row marked `can't receive cross-session messages` refuses it before it leaves; the result begins `Not sent`.

### Reply to the address on the message

- Reply to the sender named on the incoming message.
- Do not reuse a name or id remembered from earlier: a restarted session can answer to a different identifier, and an old id can point at a finished agent.

### One task per message

| One message, three tasks | Three messages, three ids |
|---|---|
| One reply mixes three answers | One reply per task |
| One task gets forgotten | Each task has its own acknowledgement |
| Hard to track | Each id is a row in the tracker |

### Writing the message

#### Before

```text
can you look at the export thing
```

#### After

```text
From the chief, task export-01: the nightly export failed at 02:10 with a
timeout. Find the cause and fix it with a regression test. Reply "taken
export-01" when you start, and the commit plus test output when done.
```

The second needs no follow-up question, and its reply can be matched to the task.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `/list-agents` is not recognised | Claude Code is older than the required version |
| A local session is missing from the list | It runs in bare mode, in a container, or across the WSL 2 and Windows boundary |
| A session on another machine is missing | That session or this one is not connected to Remote Control |
| A send did not arrive | The receiver's `crossSessionInbound` holds or refuses it, or a permission deny rule removed `SendMessage` |
| A held message vanished | Its approval dialog expired after `dialogExpiry` |

## Agent teams

Agent teams are a separate, experimental feature for a group of agents working one problem together.

| | Cross-session messaging | Agent teams |
|---|---|---|
| Who | Sessions that already exist and live independently | A lead session and the teammates it spawns |
| Turning it on | On by default | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| Lifetime | Hours or days | The lead session's lifetime; in-process teammates are not restored on resume |
| Coordination | Messages only | Messages, plus a shared task list with dependencies |
| Nesting | Any session can message any listed one | One team per session; teammates cannot spawn their own teammates |
| Cost | One session's tokens each | Each teammate is a separate instance; tokens scale with team size |

The agent teams docs suggest starting with three to five teammates, and they add hooks that enforce quality gates at team events: `TeammateIdle`, `TaskCreated` and `TaskCompleted`, each of which can exit with code 2 to keep a teammate working or block a task change.

## Sources

- Cross-session messaging: [Message your other Claude Code sessions (Claude Code docs)](https://code.claude.com/docs/en/cross-session-messaging.md), verified 2026-11-09.
- Agent teams: [Orchestrate teams of Claude Code sessions (Claude Code docs)](https://code.claude.com/docs/en/agent-teams.md), verified 2026-11-09.
- Remote Control requirements: [Remote Control (Claude Code docs)](https://code.claude.com/docs/en/remote-control.md), verified 2026-11-08.

{% include page_nav.html %}
