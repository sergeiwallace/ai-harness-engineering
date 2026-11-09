---
title: "Coordinating several sessions"
nav_order: 19
has_children: true
has_toc: false
permalink: /sections/09-session-coordination/
---

{% include page_nav.html %}

# Coordinating several sessions

As of 2026-11-09.

Running several agent sessions at once multiplies output, and it multiplies the places you have to watch. A coordinator session gives you one place to talk to. It routes each request to a worker session, reaches that worker over a channel that works, checks the evidence that comes back, and reports only what needs you.

![A sequence diagram of one task. You ask the chief to fix the nightly report. The chief finds the worker is not listed by ListAgents, so in an optional fallback fragment it sends the task through Agent Mail, which delivers it to the worker, and the worker replies "taken export-01". The worker commits the fix to the git repo and reports done. The chief checks the commit in the repo, sees the commit exists and the test passes, and only then tells you "fixed, merged".]({{ '/assets/images/09-session-coordination/coordinator-routes-a-task.svg' | relative_url }}){: style="width:100%"}

*Figure: one task through a coordinator, from request to checked result. Made with archify.*

Stop 7 on the talk path, about 2 minutes.

Pages in this section, in reading order:

- This overview: the problem, the coordinator pattern, messaging, shared actions and setup.
- [Coordinator and chief-of-staff sessions]({{ '/sections/09-session-coordination/coordinator-sessions.html' | relative_url }}): the roles, the coordinator's working cycle, and its failure modes.
  - [Owning shared actions]({{ '/sections/09-session-coordination/shared-action-ownership.html' | relative_url }}): one owner per machine for merges, pushes and installs, and locks for the rest.
- [Message channels and fallbacks]({{ '/sections/09-session-coordination/message-channels.html' | relative_url }}): the channel ladder, probing a target, and confirming delivery.
  - [SendMessage and ListAgents]({{ '/sections/09-session-coordination/sendmessage-and-listagents.html' | relative_url }}): Claude Code's built-in cross-session messaging, its settings and limits, and agent teams.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Why coordination becomes a problem

One session is a conversation you watch. Past two or three, the agents produce work faster than one person can follow it.

| Sessions | What you do | Where the time goes |
|---|---|---|
| 1 | Watch it and answer as it asks | The task |
| 2 to 3 | Switch between terminal tabs | Some switching |
| 4 or more | Visit each tab to find out what happened | Switching, missed questions, collisions |

What grows with the number of sessions:

- Places to check: one tab per session, each finishing at its own time.
- Questions to answer: each session asks its own, and waits until you see it.
- Collisions: two sessions push, install or run a heavy test suite at the same moment.

<!-- IMAGE NEEDED: a terminal window with many agent sessions open in tabs or panes; search: tmux many panes, terminal multiplexer screenshot, multiple terminal sessions -->

### Example: four sessions, one afternoon

You start four sessions: one fixes a bug, one writes a design doc, one migrates a test suite, one reviews a pull request. An hour later:

| Time | Bug fix | Design doc | Test migration | Review |
|---|---|---|---|---|
| 14:05 | Wants to push | Writing | Writing | Done, idle |
| 14:06 | Pushes | Has a question for you | Wants to push | Idle |
| 14:07 | Push lands | Waiting for you | Push rejected: conflict | Idle |
| 14:45 | Idle | Still waiting: tab not in front of you | Retried with an unreviewed rebase | Idle |

You learn about each event only by visiting its tab.

> Each session did its own work correctly. The rejected push, the unreviewed rebase and the forty-minute wait came from nobody tracking all four sessions.
{: .important }

## Ways to run several agents

Coordinating separate sessions is one of four ways Claude Code runs more than one agent. They differ in who holds the plan and how long the agents live.

| Way | Who decides what runs next | Agents live for | Talk to each other | Covered in |
|---|---|---|---|---|
| Sub-agents | The main agent, turn by turn | One task | Return a result to the caller | [Delegation]({{ '/sections/08-delegation/' | relative_url }}) |
| Agent teams (experimental) | A lead session | One session's lifetime | Directly, through a shared task list and mailboxes | [SendMessage and ListAgents]({{ '/sections/09-session-coordination/sendmessage-and-listagents.html' | relative_url }}#agent-teams) |
| Separate sessions with cross-session messaging | You, or a coordinator session | Hours or days | `SendMessage` between sessions | This section |
| Dynamic workflows | A script | One run | Results pass through the script | [Orchestration]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}) |

This section is about the third row: long-running sessions you start yourself, often in separate worktrees or on separate machines.

## The coordinator pattern

A coordinator session takes requests from the person, hands them to other sessions or agents, and reports back. It is an ordinary session with a different job.

- What you do: talk to one session.
- What it does: route, send, track, check and report.
- What it does not do: the leaf work. Its context window stays free for the picture of all the sessions, for the same reason a lead agent delegates in [delegation]({{ '/sections/08-delegation/' | relative_url }}).

### Giving it the role

Two layers set the role up. The context file describes the job, and the session's permissions enforce the part that matters most: it cannot edit files.

```markdown
## Role: coordinator
Route each request to the session that already has its context, or start one.
Ask workers for evidence, check it, then report.
```

```bash
# start the coordinator with edits denied, so the "do not do the work" rule holds
claude --name coordinator \
  --settings '{"permissions":{"deny":["Edit","Write"]}}'
```

The deny rule does the enforcing. The context-file line explains the role to the model and to anyone reading the setup. [A systems approach]({{ '/sections/02-ai-harnesses/' | relative_url }}#a-systems-approach-to-harness-engineering) covers this layering in general.

### What a coordinator does

| Step | The coordinator | What you see |
|---|---|---|
| 1. Route | Picks the target: an existing session that already has the context, a new session, or a sub-agent | Nothing |
| 2. Send | Sends the work with enough context to act on, as a [brief]({{ '/sections/07-templates-and-stubs/' | relative_url }}) does, naming the evidence wanted back | Nothing |
| 3. Track | Keeps the in-flight list in a tracker, so it survives [compaction]({{ '/sections/11-context-window/compaction.html' | relative_url }}) | Nothing |
| 4. Check | Verifies the commit, the test output or the page before believing "done" | Nothing |
| 5. Report | Brings you only a decision, a blocker or a finished result | One line |

The full cycle, with its failure modes, is on the [coordinator page]({{ '/sections/09-session-coordination/coordinator-sessions.html' | relative_url }}).

### Example: routing a request

You tell the coordinator "the nightly report job is failing again".

1. The coordinator picks a target.
   - It checks which session last worked on the report job.
   - It finds one still running with that code in its context.
2. It sends that session the request, asking for the commit and the test output back.
3. Twenty minutes later the session reports a fix with its test output.
4. The coordinator checks the evidence.
   - The commit exists on the branch.
   - The named test passed in the output.
5. It tells you in one line: fixed, merged, with the commit.

You never opened the worker's tab. The figure at the top of this page shows the same flow with a fallback channel.

### Tiers on several machines

With more than one machine, the machines also need to agree on who does what. My setup has two tiers:

| Tier | Sessions | Job |
|---|---|---|
| Top coordinator | One | Takes my requests, picks the machine, reports results and blockers |
| Chief of staff | One per machine | Routes work to the sessions on its machine and owns that machine's shared actions |
| Workers | Any number | Do the tasks; ask their chief before a shared action |

The top coordinator sends work to a machine's chief, never to a worker on another machine directly.

### When you do not need one

| Situation | Simpler option |
|---|---|
| Two sessions on unrelated work | Check each yourself; a shared notes file if they touch the same repo |
| One task split into parallel parts | [Sub-agents]({{ '/sections/08-delegation/' | relative_url }}) from one session |
| Many identical steps over many files | A [dynamic workflow]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}) |
| A short group effort with discussion | An agent team |

## How sessions message each other

In Claude Code, two tools carry messages between sessions. Claude calls them itself; you ask in plain words ("tell the payments session the schema changed").

| Tool or command | What it does |
|---|---|
| `ListAgents` | Lists the sub-agents, teammates and sessions this session can message |
| `SendMessage` | Sends one of them a plain-text message |
| `/list-agents` (also `/peers`) | Shows you the same list, with this session's own name on the first line |
| `@name` in a prompt | Names the target session for Claude, from a typeahead of live sessions |
| `/rename <name>` or `claude --name <name>` | Gives a session the name other sessions use to reach it |

How far a message reaches:

- On the same machine: every provider, including API keys and Amazon Bedrock. Messages travel over a per-session socket and never through Anthropic's servers.
- To your sessions on other machines and in the cloud: only from a session connected to Remote Control, which needs a claude.ai sign-in. It does not work with an API key or on Bedrock and the other cloud providers.

More, including inbound settings and limits, in [SendMessage and ListAgents]({{ '/sections/09-session-coordination/sendmessage-and-listagents.html' | relative_url }}).

### Example: a message and its reply

A simplified transcript from a coordinator session:

```text
> /list-agents
  coordinator                              (this session)
  report-fixer    running   ~/repos/reports
  design-writer   idle      ~/repos/design

> Tell @report-fixer the nightly job failed at 02:10 with a timeout in the
  export step; ask for a fix with a regression test, and the commit back.

  ... later ...
  › Message from @report-fixer: Fixed: the export query lost its index ...
```

The expanded message reads "Fixed: the export query lost its index after last week's migration. Commit 4e1a9c2, new test test_export_uses_index passes."

### When the built-in channel is not enough

Separate messaging systems fill the gaps. Use them as ordered fallbacks behind one primary channel.

| Gap | Fallback that covers it |
|---|---|
| Another machine without Remote Control | A shared mail service such as Agent Mail, or a git repository used as an inbox |
| A different tool on the other side, such as Codex | A mail service both can reach |
| A broadcast to several sessions | A message board |
| The built-in channel refuses or holds the message | The next rung on the ladder |

![A sequence diagram. The sender probes ListAgents through the native channel and learns the target is absent. In an optional fragment it probes the target's Agent Mail identity and learns it was last seen three hours ago. It commits the message to the git inbox; the target pulls its inbox and replies taken.]({{ '/assets/images/09-session-coordination/channel-ladder.svg' | relative_url }}){: style="width:100%"}

*Figure: walking the ladder: each probe asks whether the target will read it. Made with archify.*

[Message channels and fallbacks]({{ '/sections/09-session-coordination/message-channels.html' | relative_url }}) covers the ladder, probes and delivery checks.

## Owning shared actions

Some actions are safe in parallel and some are not.

| Safe in parallel | Unsafe in parallel | What goes wrong |
|---|---|---|
| Editing files in separate worktrees | Merging to `main` | Two merges pass alone and break together |
| Reading and searching | Pushing to one shared branch | A rejected push, then an unreviewed rebase |
| Running a small test file | A heavy test suite on a shared machine | Every session's tests time out |
| | Reinstalling a shared tool | A half-written install while another session runs it |

One session per machine can own the unsafe actions. The other sessions send it a request, and it runs them one at a time. A hook or a lock can enforce this, so a worker that forgets the rule is refused at the moment it tries.

### Example: two pushes become a queue

1. The migration session and the bug-fix session both finish. Neither pushes.
2. Each sends the chief "ready to merge" with its branch name and test evidence.
3. The chief merges the bug fix.
   - It runs the tests on the new `main`: green.
4. It rebases the migration onto the new `main` and merges it.
   - It runs the tests again: green.
5. It confirms to both workers.

![A sequence diagram. Worker A and worker B both ask the chief to merge. Inside a critical fragment the chief merges A into main, runs the tests, gets green and confirms to A; then merges B, runs the tests again, gets green and confirms to B.]({{ '/assets/images/09-session-coordination/merge-owner.svg' | relative_url }}){: style="width:100%"}

*Figure: two merge requests become a queue under one owner. Made with archify.*

[Owning shared actions]({{ '/sections/09-session-coordination/shared-action-ownership.html' | relative_url }}) covers which actions to own, locks for the ones that are not merges, and how to keep the owner from becoming a bottleneck.

## Setting it up

A minimal setup on one machine:

1. Start each worker with a name you will recognise.
   - `claude --name report-fixer`, or `/rename report-fixer` inside a running session.
2. Start the coordinator with edits denied (see [giving it the role](#giving-it-the-role)).
3. In the coordinator, run `/list-agents` and check every worker is listed.
   - A session missing from the list may be on an older Claude Code version, in bare mode, or in a container the coordinator cannot reach.
4. Give each task a short id, and ask for an acknowledgement and evidence in every message.
5. For long tasks, ask to be told when a worker goes idle ("tell me when the migration session finishes").
   - Claude subscribes with `SendMessage`'s `notify_when_idle` input. One notice arrives when that session next goes idle or exits, or the subscription lapses after 12 hours.
6. For a headless worker started with `claude -p`, pass `"crossSessionInbound": "accept"` in its `--settings`, so messages are delivered without an approval dialog.

## Examples from my own harness

- A coordination layer built on firstmate, an open-source project by GitHub user kunchenguid (<https://github.com/kunchenguid/firstmate>). I use my own public fork with customizations for my setup (<https://github.com/sergeiwallace/firstmate>).
  - One firstmate session takes my requests and supervises a chief-of-staff session on each machine.
  - Each chief routes work to the sessions on its machine and runs pushes and merges one at a time.
  - Workers that finish a change send their chief "ready to merge" instead of merging.
- A fixed order of message channels.
  - `SendMessage` first, then Agent Mail (<https://github.com/Dicklesworthstone/mcp_agent_mail>), then Buzz (<https://github.com/block/buzz>), then a git-based inbox for other machines.
  - Each channel is probed for the target before use.

To start: two named sessions and one shared notes file. Each session appends what it is doing and what it needs, and you read one file instead of two tabs.

## Pitfalls and lessons

| Pitfall | What it looks like | What helps |
|---|---|---|
| Wrong channel | A message goes where the target does not read, and the sender waits | One primary channel, ordered fallbacks |
| Too many channels | Each adds its own probes, fallbacks and failures | Add a channel only for reach the others lack |
| A working coordinator | It does leaf work, fills its context and stops answering | Deny it edit tools; the role line alone is not enough |
| "Sent" read as "received" | A task is lost or sent twice | Count delivery only when the target replies |
| Unnamed sessions | The list shows names Claude Code picked, and messages go to the wrong one | Name every session with `--name` or `/rename` |

## Sources

- Cross-session messaging (`ListAgents`, `SendMessage`, `/list-agents`, naming, reach, `notify_when_idle`, `crossSessionInbound`): <https://code.claude.com/docs/en/cross-session-messaging.md>, verified 2026-11-09.
- Agent teams: <https://code.claude.com/docs/en/agent-teams.md>, verified 2026-11-09.
- Remote Control requirements: <https://code.claude.com/docs/en/remote-control.md>, verified 2026-11-08.
- firstmate: <https://github.com/kunchenguid/firstmate>, verified 2026-11-08. My fork: <https://github.com/sergeiwallace/firstmate>.
- Agent Mail: <https://github.com/Dicklesworthstone/mcp_agent_mail>. Buzz: <https://github.com/block/buzz>.

{% include page_nav.html %}
