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

Fires once on exit. It cannot change the session, so use it for cleanup and records: write the session's cost to a log, release a lock the session held, mark its tasks for review.

## Prompt events

### UserPromptSubmit

Runs before the model sees your prompt. It can add context or refuse the prompt.

#### Worked example

A hook watches for the words "deploy" and "release" in a prompt and, when it sees one, adds a two-line reminder of the release checklist's location. The checklist stays out of the always-loaded context file and appears exactly when it is relevant.

## Tool events

### PreToolUse

The most used event. It sees the tool name and its full input before anything runs, and it can allow, ask or deny. [Guard hooks]({{ '/sections/02-ai-harnesses/guard-hooks.html' | relative_url }}) covers it in depth.

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

Fires when the harness is about to show a permission prompt. A hook here can approve or refuse by code, which suits rules too subtle for a [permission pattern]({{ '/sections/02-ai-harnesses/permissions-and-settings.html' | relative_url }}).

## Turn events

### Stop

Fires when the agent decides it is done. A hook that returns `"decision": "block"` with a reason sends it back to work.

#### Worked example

A Stop hook runs the fast test suite. If any test fails, it blocks the stop with the failing test names. The agent cannot end its turn claiming success while the tests are red.

> A Stop hook that always blocks traps the agent in a loop. The event input says whether a stop hook already sent the agent back once; check it and let the second stop through.
{: .warning }

### SubagentStop

The same, for a sub-agent. My harness uses this kind of check on a sub-agent's report: if a required field is missing (the head commit, the checks run, what is unfinished), the sub-agent is sent back to add it before its launcher collects it.

## Context events

### PreCompact

Fires before the harness summarises the context. Detail in the context is about to be lost, so this is the last chance to write it somewhere durable. See [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}#compaction).

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
