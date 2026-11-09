---
title: "Coordinator and chief-of-staff sessions"
parent: "Coordinating several sessions"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/09-session-coordination/detail.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Coordinator and chief-of-staff sessions

A coordinator session is a long-running session whose job is routing work to other sessions and checking what comes back. This page covers the roles in a coordination layer, the coordinator's working cycle, what it keeps in its context and what it keeps in files, how it checks reports, the ways it goes wrong, and what to do when the coordinator itself stops. The [section overview]({{ '/sections/09-session-coordination/' | relative_url }}) is the short version. A sub-topic page covers [owning shared actions]({{ '/sections/09-session-coordination/shared-action-ownership.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Roles

A coordination layer has three kinds of session. The role names are my own.

| Role | Talks to | Does | Does not do |
|---|---|---|---|
| Top coordinator | You, and the chiefs | Takes your requests, picks the machine, reports results and blockers | Edit code, run builds |
| Chief of staff (one per machine) | The top coordinator, and the workers on its machine | Routes work on its machine, owns shared actions, checks workers | Long feature work |
| Worker | Its chief | The actual task: code, docs, research, review | Push to shared branches, install shared tools |

On a single machine the top coordinator and the chief can be the same session. The split matters once two machines need to agree on who does what.

### Why a separate session at all

Coordination and leaf work need different views, and one context window holds them badly together.

| | Coordination | Leaf work |
|---|---|---|
| View | Wide and shallow: what is in flight, blocked, finished | Narrow and deep: this file, this test, this error |
| Context contents | Task list, session list, short reports | File contents, stack traces, test output |
| Fills the window | Slowly | Quickly |

A session doing both compacts sooner, and while it reads files it stops answering other sessions. The same reasoning drives a lead agent to hand search and review to sub-agents in [delegation]({{ '/sections/08-delegation/' | relative_url }}).

### The job description

Two layers set up the role. The context file describes the job; the session's permissions enforce the part that must hold.

```markdown
## Role: chief of staff for this machine

- Route each request to the session that already has its context, or start one.
- You own merges, pushes to main and installs on this machine. Workers send
  "ready to merge" with a branch name; merge one at a time.
- Report to the top coordinator: done (with evidence), blocked (with what
  unblocks it), or needs a decision (with options).
```

```bash
claude --name chief \
  --settings '{"permissions":{"deny":["Edit","Write"]}}'
```

| Rule | Where it lives | Why there |
|---|---|---|
| Do not edit project files | A permission deny rule | A model under pressure skips a written rule; a denied tool cannot be called |
| Route, track, check, report | The context file | Judgment about how to route cannot be enforced by a tool list |
| Merge one at a time | The chief's merge script, or a lock | A script that takes a lock serialises merges whatever the model decides |

The chief still needs `Bash` for `git` and its merge script, so the deny list names only the edit tools.

## The working cycle

![A sequence diagram of one coordinator cycle. You ask the chief for a fix; the chief records the task in the tracker and sends it to the worker with the proof it wants back; the worker replies taken. In an optional fragment the worker asks a question, the chief forwards it to you, you answer, and the chief relays the answer. The worker reports done, the chief checks the commit, closes the task in the tracker and reports the result to you.]({{ '/assets/images/09-session-coordination/coordinator-cycle.svg' | relative_url }}){: style="width:100%"}

*Figure: one coordinator cycle, from request to closed task. Made with archify.*

### 1. Take the request

- Restate the request in one line, so a misunderstanding shows up at once.
- Record it in the task tracker with an id.
  - A request that lives only in the conversation is lost at the coordinator's next compaction ([context window and memory]({{ '/sections/11-context-window/' | relative_url }})).

### 2. Pick a target

Three options, in the order I try them:

1. An existing session that already has the relevant code in its context. It needs the least briefing.
2. A new session, when no existing one fits or the existing ones are busy.
3. A sub-agent of the coordinator, for a short self-contained check such as "does this branch still build".

### 3. Send the task with the evidence wanted back

The message says what to do and what proof to return. Asking for the proof up front makes the check in step 5 cheap.

```text
Task export-01: the nightly export times out. Fix it with a regression test.
Reply "taken export-01" when you start. When done, reply with the commit hash
and the output of the new test.
```

### 4. Track

The coordinator keeps one row per task in flight, in the tracker:

| Task | Session | Sent | Last word | Expected by |
|---|---|---|---|---|
| export-01 | report-fixer | 14:02 | "taken" at 14:03 | 15:00 |
| docs-07 | design-writer | 14:10 | none | 14:40 |

- A task with no acknowledgement after a few minutes gets a delivery check.
- A task past its expected time gets a status question.
- For a long task on the same machine, the coordinator can ask for an idle notice instead of polling: Claude Code's `SendMessage` takes a `notify_when_idle` input, and one notice arrives when that session next goes idle or exits (the subscription lapses after 12 hours).

### 5. Check the report

A worker's report of "done" goes on to you only after the coordinator checks it, the same check as in [delegation]({{ '/sections/08-delegation/' | relative_url }}) done one level up.

| Report says | Coordinator checks | Command |
|---|---|---|
| "Committed 4e1a9c2" | The commit exists on the worker's branch | `git -C <worktree> branch --contains 4e1a9c2` |
| "Test passes" | The test passes when the coordinator runs it | `pytest tests/test_export.py::test_export_uses_index -q` |
| "Deployed" | The page or endpoint answers | `curl -fsS https://<service>/health` |
| "Merged" | `main` contains the change | `git merge-base --is-ancestor 4e1a9c2 origin/main && echo merged` |

Running the check beats reading the worker's pasted output: pasted output can come from an earlier run.

### 6. Report to you

| Kind | What the coordinator sends | Example |
|---|---|---|
| Done | One line with the evidence | "export-01 fixed, merged, commit 4e1a9c2" |
| Blocked | What is blocked and what unblocks it | "docs-07 needs the API owner's name" |
| Decision | The options and a recommendation | "Restore the index, or rewrite the query? I recommend the index: one migration, no query changes" |

## What lives in context and what lives in files

A coordinator runs for hours or days, so it compacts many times. Anything it needs after a compaction has to be in a file.

| Kept in files | Kept in context only |
|---|---|
| Open requests and who has them (task tracker) | The current conversation with you |
| Which sessions exist, on which machine | The worker it is talking to right now |
| Decisions you made, with dates | Its reasoning about the next step |
| Standing rules for its role (context file) | |

### Example: surviving a compaction

A chief has six tasks in flight when its context fills.

1. Before compacting, it writes a state note.
   - Each task, the session that has it, and its last status.
2. It compacts.
3. After compaction it rebuilds its picture.
   - It reads the note and the tracker.
   - It runs `ListAgents` to confirm which sessions still exist.
4. One worker finished during the compaction. The chief sees its message waiting, checks the commit, and reports it.

## Several machines

With two or more machines, each machine has its own chief. The top coordinator sends work to a machine's chief, never directly to a worker on another machine.

- The chief knows its machine's state: which sessions are busy, whether a heavy test run is in progress, whether a merge is underway.
- Shared actions on a machine have one owner. A worker taking orders from two places would break that.

| Link between machines | Needs |
|---|---|
| Claude Code's built-in messaging | Both sessions connected to Remote Control, with a claude.ai sign-in |
| A mail service, board or git inbox | Each machine able to reach it; see [message channels]({{ '/sections/09-session-coordination/message-channels.html' | relative_url }}) |

### Example: picking a machine

You ask for a long data-processing job and a quick docs fix.

1. The top coordinator checks each machine's chief for capacity and context.
   - Machine A is idle.
   - Machine B already runs the docs session.
2. It sends the job to machine A's chief and the docs fix to machine B's chief.
3. Each chief routes inside its own machine.
   - A's chief starts a new session for the job.
   - B's chief sends the fix to the existing docs session.

## Failure modes

### The coordinator starts working

- What happens: a small fix looks quicker to do than to route, so the coordinator does it.
  - Its context fills with file contents and it compacts sooner.
  - While it works, it stops answering other sessions.
- What helps:
  - Deny the coordinator the edit tools (see [the job description](#the-job-description)), so the shortcut is unavailable.
  - The role line in its context file, as a second layer that explains why.

### Relaying without checking

- What happens: the coordinator forwards "done" without looking at the evidence, and a worker's mistake becomes its report.
  - Agents report work they did not do.
- What helps: ask for evidence in the task message (step 3), and check it (step 5) before reporting.

### Waiting on a message that never arrived

- What happens: the task goes to a session that no longer exists, or the receiver holds it, and the coordinator waits.
  - On the built-in channel, a message held for approval is dropped after the receiver's `dialogExpiry`, five minutes by default.
- What helps:
  - Check the list before sending.
  - Ask for an acknowledgement, and count delivery only when it arrives.
  - Give each task an expected time, and check on it when the time passes.
  - Start unattended workers with `crossSessionInbound` set to `accept`, so messages are not held.

### Asking you too much

- What happens: every question goes up to you, and you are back to watching every session.
- What helps: give the coordinator authority over routine choices, and send you only what it may not decide.

| Coordinator decides | You decide |
|---|---|
| Which session, which order | Scope changes |
| Retry or not | Anything destructive or outward-facing |
| Which channel | Design decisions |

[Human gates]({{ '/sections/14-human-gates/' | relative_url }}) covers where that line sits.

## When the coordinator stops

Workers keep running when the coordinator exits or crashes; nothing is lost if the coordinator's state is in files.

1. Start a new coordinator with the same name and settings.
2. Rebuild its picture.
   - Read the tracker's in-flight rows and the last state note.
   - Run `/list-agents` and match each in-flight task to a live session.
3. Ask each worker with an in-flight task for its status.
4. Re-send only the tasks whose session is gone, with their original ids.

## Examples from my own harness

- A coordination layer built on [firstmate](https://github.com/kunchenguid/firstmate), an open-source project by GitHub user kunchenguid. I use [my own public fork](https://github.com/sergeiwallace/firstmate) with customizations for my setup.
  - One firstmate session takes my requests and supervises a chief-of-staff session on each machine.
- Each chief enrols the sessions it finds on its machine, routes work to them, and runs pushes and merges one at a time.
- A fleet briefing, generated from live state, that lists each session, what it drives, whether it is busy or idle, and what needs me.

To start: one coordinator session started with edits denied and a context file that says "route, track, check, report", plus one named worker.

## Sources

- Cross-session messaging (`ListAgents`, `SendMessage`, `notify_when_idle`, `crossSessionInbound`, `dialogExpiry`): [Message your other Claude Code sessions (Claude Code docs)](https://code.claude.com/docs/en/cross-session-messaging.md), verified 2026-11-09.
- Permission deny rules and `--settings`: [Permissions (Claude Code docs)](https://code.claude.com/docs/en/permissions.md).
- firstmate: [kunchenguid/firstmate on GitHub](https://github.com/kunchenguid/firstmate), verified 2026-11-08. My fork: [sergeiwallace/firstmate on GitHub](https://github.com/sergeiwallace/firstmate).

{% include page_nav.html %}
