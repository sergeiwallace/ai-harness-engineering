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

Once you run more than two or three agent sessions, the limit is you: how many tabs you check and how many questions you answer. A coordinator session fixes that. You talk to one session; it routes each request to the right worker, reaches that worker over the best channel that works, checks the evidence that comes back, and tells you only what needs you. One owner per machine runs the actions that must not happen twice at once, such as merges and installs.

![A sequence diagram of one task. You ask the chief to fix the nightly report. The chief finds the worker is not listed by ListAgents, so in an optional fallback fragment it sends the task through Agent Mail, which delivers it to the worker, and the worker replies "taken export-01". The worker commits the fix to the git repo and reports done. The chief checks the commit in the repo, sees the commit exists and the test passes, and only then tells you "fixed, merged".]({{ '/assets/images/09-session-coordination/coordinator-routes-a-task.svg' | relative_url }}){: style="width:100%"}

*Figure: one task through a coordinator, from request to checked result. Made with archify.*

Stop 7 on the talk path, about 2 minutes. This overview covers why several sessions need coordinating, the coordinator pattern, how sessions message each other, and who owns shared actions. The topic pages go deeper:

- [Coordinator and chief-of-staff sessions]({{ '/sections/09-session-coordination/coordinator-sessions.html' | relative_url }}): the roles, what a coordinator does each cycle, and how it fails.
- [Owning shared actions]({{ '/sections/09-session-coordination/shared-action-ownership.html' | relative_url }}): one owner per machine for pushes, merges and installs.
- [Message channels and fallbacks]({{ '/sections/09-session-coordination/message-channels.html' | relative_url }}): the channel ladder, probing, delivery checks.
- [SendMessage and ListAgents]({{ '/sections/09-session-coordination/sendmessage-and-listagents.html' | relative_url }}): Claude Code's built-in cross-session messaging, and agent teams.

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

### Example: four sessions, one afternoon

You start four sessions: one fixes a bug, one writes a design doc, one migrates a test suite, one reviews a pull request. An hour later:

| Time | Bug fix | Design doc | Test migration | Review |
|---|---|---|---|---|
| 14:05 | Wants to push | Writing | Writing | Done, idle |
| 14:06 | Pushes | Has a question for you | Wants to push | Idle |
| 14:07 | Push lands | Waiting for you | Push rejected: conflict | Idle |
| 14:45 | Idle | Still waiting: tab not in front of you | Retried with an unreviewed rebase | Idle |

You learn about each event only by visiting its tab.

> Each session did its own work correctly. The two failed pushes and the forty-minute wait came from nobody tracking all four sessions.
{: .important }

## The coordinator pattern

A coordinator session takes requests from the person, hands them to other sessions or agents, and reports back. It is an ordinary session with a different job description in its context file.

- What you do: talk to one session.
- What it does: route, send, track, check and report.
- What it does not do: the leaf work. Its context window stays free for the picture of all the sessions, for the same reason a lead agent delegates in [delegation]({{ '/sections/08-delegation/' | relative_url }}).

A role description of a few lines is enough to start:

```markdown
## Role: coordinator
Route each request to the session that already has its context, or start one.
Do not edit project files. Ask workers for evidence, check it, then report.
```

### What a coordinator does

| Step | The coordinator | What you see |
|---|---|---|
| 1. Route | Picks the target: an existing session that already has the context, a new session, or a sub-agent | Nothing |
| 2. Send | Sends the work with enough context to act on, as a [brief]({{ '/sections/07-templates-and-stubs/' | relative_url }}) does, naming the evidence wanted back | Nothing |
| 3. Track | Keeps the in-flight list in a tracker, so it survives [compaction]({{ '/sections/11-context-window/compaction.html' | relative_url }}) | Nothing |
| 4. Check | Verifies the commit, the test output, the page, before believing "done" | Nothing |
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

The top coordinator sends work to a machine's chief, never to a worker on another machine directly. [Coordinator and chief-of-staff sessions]({{ '/sections/09-session-coordination/coordinator-sessions.html' | relative_url }}) covers the roles in depth.

## How sessions message each other

In Claude Code, two tools carry messages between sessions:

| Tool | What it does | Reach |
|---|---|---|
| `ListAgents` | Lists running agents and sessions this session can message | This machine, and other machines through Remote Control |
| `SendMessage` | Sends one of them a message, which arrives as a new turn | The same |

- On the same machine this works with any provider, including API billing.
- Reaching another machine needs Remote Control, which needs a claude.ai subscription.
- `/list-agents` is the slash command a person types to see the same list.

More in [SendMessage and ListAgents]({{ '/sections/09-session-coordination/sendmessage-and-listagents.html' | relative_url }}).

### Example: a message and its reply

A simplified transcript from a coordinator session:

```text
> ListAgents
  report-fixer    running   (this machine)
  design-writer   idle      (this machine)

> SendMessage to=report-fixer
  "The nightly report job failed at 02:10 with a timeout in the export step.
   Find the cause, fix it with a regression test, and reply with the commit
   and the test output."

  ... later ...
  [message from report-fixer]
  "Fixed: the export query lost its index after last week's migration.
   Commit 4e1a9c2, new test test_export_uses_index passes."
```

### When the built-in channel is not enough

Separate messaging systems fill the gaps. Use them as ordered fallbacks behind one primary channel.

| Gap | Fallback that covers it |
|---|---|
| Another machine without Remote Control | A shared mail service such as Agent Mail, or a git repository used as an inbox |
| A different tool on the other side, such as Codex | A mail service both can reach |
| A broadcast to several sessions | A message board |
| The built-in channel is down | The next rung on the ladder |

[Message channels and fallbacks]({{ '/sections/09-session-coordination/message-channels.html' | relative_url }}) covers the ladder, probes and delivery checks.

![A sequence diagram. The sender probes ListAgents through the native channel and learns the target is absent. In an optional fragment it probes the target's Agent Mail identity and learns it was last seen three hours ago. It commits the message to the git inbox; the target pulls its inbox and replies taken.]({{ '/assets/images/09-session-coordination/channel-ladder.svg' | relative_url }}){: style="width:100%"}

*Figure: walking the ladder: each probe asks whether the target will read it. Made with archify.*

## Owning shared actions

Some actions are safe in parallel and some are not.

| Safe in parallel | Unsafe in parallel | What goes wrong |
|---|---|---|
| Editing files in separate worktrees | Merging to `main` | Two merges pass alone and break together |
| Reading and searching | Pushing to one shared branch | A rejected push, then an unreviewed rebase |
| Running a small test file | A heavy test suite on a shared machine | Every session's tests time out |
| | Reinstalling a shared tool | A half-written install while another session runs it |

One session per machine can own the unsafe actions. The other sessions send it a request, and it runs them one at a time.

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

## Examples from my own harness

- A coordination layer built on firstmate, an open-source project by GitHub user kunchenguid (<https://github.com/kunchenguid/firstmate>). I use my own public fork with customizations for my setup (<https://github.com/sergeiwallace/firstmate>).
  - One firstmate session takes my requests and supervises a chief-of-staff session on each machine.
  - Each chief routes work to the sessions on its machine and runs pushes and merges one at a time.
  - Workers that finish a change send their chief "ready to merge" instead of merging.
- A fixed order of message channels.
  - `SendMessage` first, then Agent Mail (<https://github.com/Dicklesworthstone/mcp_agent_mail>), then Buzz (<https://github.com/block/buzz>), then a git-based inbox for other machines.
  - Each channel is probed for the target before use.

To start: two sessions and one shared notes file. Each session appends what it is doing and what it needs, and you read one file instead of two tabs.

## Pitfalls and lessons

| Pitfall | What it looks like | Fix |
|---|---|---|
| Wrong channel | A message goes where the target does not read, and the sender waits | One primary channel, ordered fallbacks |
| Too many channels | Each adds its own probes, fallbacks and failures | Add a channel only for reach the others lack |
| A working coordinator | It does leaf work, fills its context and stops answering | Keep its job description narrow |
| "Sent" read as "received" | A task is lost or sent twice | Count delivery only when the target replies |

## Sources

- firstmate: <https://github.com/kunchenguid/firstmate>, verified 2026-11-08. My fork: <https://github.com/sergeiwallace/firstmate>.
- `ListAgents` and `SendMessage`: <https://code.claude.com/docs/en/cross-session-messaging.md>, verified 2026-11-08.
- Remote Control requirements: <https://code.claude.com/docs/en/remote-control.md>, verified 2026-11-08.
- Agent Mail: <https://github.com/Dicklesworthstone/mcp_agent_mail>. Buzz: <https://github.com/block/buzz>.

{% include page_nav.html %}
