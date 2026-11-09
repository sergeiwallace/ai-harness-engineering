---
title: "Hook lifecycle events"
parent: "Harness hooks"
grand_parent: "AI harnesses"
nav_order: 1
---

{% include page_nav.html %}

# Hook lifecycle events

As of 2026-11-09. A sub-topic of [Harness hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}), in [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}).

Each hook event is a moment in the session where the harness pauses and asks your code what to do. Picking the right event is most of the design of a hook: the same check can be useless at one event and decisive at another.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The events in order

![Where hooks fire in a session.]({{ '/assets/images/02-ai-harnesses/hook-lifecycle.svg' | relative_url }})

| Event | Fires | Can block | Typical use |
|---|---|---|---|
| `SessionStart` | when a session opens or resumes | no | add git state, open tasks, a reminder of today's focus |
| `UserPromptSubmit` | on each prompt, before the model sees it | yes | add context for keywords, refuse prompts with secrets |
| `PreToolUse` | before each tool call | yes | guards: refuse, allow or ask |
| `PermissionRequest` | when a permission prompt would show | yes | decide prompts by code |
| `PostToolUse` | after each tool call succeeds | feeds back | lint the file just written, check test output |
| `Notification` | when the harness would notify the person | no | route the alert to a phone or chat |
| `Stop` | when the agent ends its turn | sends it back | refuse to stop while checks fail |
| `SubagentStop` | when a sub-agent ends | sends it back | check the sub-agent's report has every field |
| `PreCompact` | before the context is summarised | no | save state, write a resume note |
| `SessionEnd` | when the session closes | no | log cost, release locks |

## Session events

### SessionStart

The output of a `SessionStart` hook is added to the context, so it is a way to give every session facts that change daily without editing a context file.

```text
Git: branch feature/login, 2 files modified, 1 commit ahead of origin.
Open tasks assigned to this session: 2 (highest priority first).
Reminder: a release freeze is in effect until Friday.
```

#### Keep it short

The hook runs on every start and resume, and its output costs tokens for the rest of the session. A few lines is the right size.

### SessionEnd

Fires once on exit. It cannot change the session, so it is for cleanup and records.

- Write the session's token use and cost to a log.
- Release a lock the session held, such as a claim on a shared test runner.
- Mark the session's in-progress tasks for review, so nothing stays claimed by a session that is gone.

## Prompt events

### UserPromptSubmit

Runs before the model sees your prompt.

| It can | Example |
|---|---|
| add context | the location of the release checklist when "release" appears |
| refuse the prompt | a prompt that contains something shaped like an API key |
| record the prompt | an audit log of what was asked |

Its output is added to the context, so keep it to a few lines and fire it only when its condition matches.

## Tool events

### PreToolUse

The most used event. It sees the tool name and its full input before anything runs.

- Return `permissionDecision: "deny"` with a reason to refuse.
- Return `"allow"` to run without a prompt, or `"ask"` to show one.
- Exit 2 with a message on standard error for a simple refusal.
- [Guard hooks]({{ '/sections/02-ai-harnesses/guard-hooks.html' | relative_url }}) covers writing one in depth.

### PostToolUse

Runs after a tool succeeds, with both the input and the response. It cannot undo the action, but it can tell the model about a problem while the model still has the change in view.

```json
{
  "decision": "block",
  "reason": "ruff found 2 errors in src/api.py: F401 unused import on line 3, E711 comparison to None on line 42. Fix them before continuing."
}
```

#### Worked example

After every Edit or Write to a Python file, a hook runs the linter on that file only and returns any errors. The model fixes them in the next turn, while the edit is fresh, and the commit later passes the lint gate the first time.

### PermissionRequest

Fires when the harness is about to show a permission prompt.

- A hook here can approve or refuse by code.
- It suits rules too subtle for a [permission pattern]({{ '/sections/02-ai-harnesses/permissions-and-settings.html' | relative_url }}), such as "allow `git push` only to branches this session created".
- It does not run when a rule already allowed or denied the call.

## Turn events

### Stop

Fires when the agent decides it is done. A hook that returns `"decision": "block"` with a reason sends it back to work.

#### Worked example

A Stop hook runs the fast test suite. If any test fails, it blocks the stop with the failing test names. The agent cannot end its turn claiming success while the tests are red.

> A Stop hook that always blocks traps the agent in a loop. The event input says whether a stop hook already sent the agent back once; check it and let the second stop through.
{: .warning }

### SubagentStop

The Stop event for a sub-agent.

1. A sub-agent finishes and returns its report.
2. The hook checks the report for every required field: head commit, checks run and their results, anything unfinished.
3. If a field is missing, it blocks with the list of missing fields, and the sub-agent adds them.
4. The launcher collects a complete report.

My harness uses a check of this kind on sub-agent reports.

## Context events

### PreCompact

Fires before the harness summarises the context. Detail in the conversation is about to be lost.

- Check that a resume note exists and is newer than the last commit.
- Write the current task, open decisions and git state somewhere durable.
- See [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}#compaction).

## Choosing the event

| You want to... | Use |
|---|---|
| stop an action before it happens | `PreToolUse` |
| check a result while it is fresh | `PostToolUse` |
| refuse to finish while something is wrong | `Stop` or `SubagentStop` |
| give every session today's facts | `SessionStart` |
| add context when a topic comes up | `UserPromptSubmit` |
| save state before detail is lost | `PreCompact` |
| record what a session did | `SessionEnd` |

## Sources

- Claude Code hooks reference, events and input fields: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-08.
- The worked examples are from the author's own harness, described conceptually.

{% include page_nav.html %}
