---
title: "Hook lifecycle events"
parent: "Harness hooks"
grand_parent: "What is a harness?"
nav_order: 1
redirect_from:
  - /sections/02-ai-harnesses/hook-lifecycle-events.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Hook lifecycle events

Each hook event is a moment in the session where the harness pauses and asks your code what to do. Picking the right event is most of the design of a hook: the same check can be useless at one event and decisive at another.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The events in order

![The Claude Code hook lifecycle, simplified, top to bottom. SessionStart once per session injects context. Inside the turn, once per user prompt, UserPromptSubmit validates or blocks the prompt. Inside it, the agentic loop runs once per tool call: PreToolUse can allow, deny, ask the user or rewrite the tool input; the tool executes; PostToolUse reacts to the result, gives feedback or rewrites the output; then the loop repeats. Stop can block the stop until tests pass. SessionEnd once per session does cleanup and logging.]({{ '/assets/images/02-ai-harnesses/hook-lifecycle-ran-the-builder.png' | relative_url }})

*Placeholder image: the Claude Code hook lifecycle. Source: Ran the Builder, <https://ranthebuilder.cloud/blog/agentic-coding-hooks-deterministic-ai-guardrails/>, used with credit.*

### The main events

| Event | Fires | Exit 2 does | Typical use |
|---|---|---|---|
| `SessionStart` | when a session starts, resumes, clears or compacts | nothing; output adds context | add git state, open tasks, a reminder of today's focus |
| `UserPromptSubmit` | on each prompt, before the model sees it | blocks the prompt | add context for keywords, refuse prompts with secrets |
| `PreToolUse` | before each tool call | blocks the call | guards: refuse, allow, ask or rewrite the input |
| `PermissionRequest` | when a permission prompt would show | nothing; deny through its `decision` object | decide prompts by code |
| `PostToolUse` | after a tool call succeeds | the reason is shown; the tool already ran | lint the file just written, check test output |
| `PostToolUseFailure` | after a tool call fails | | record or explain failures |
| `Notification` | when the harness would notify the person | | route the alert to a phone or chat |
| `Stop` | when the agent ends its turn | sends it back to work | refuse to stop while checks fail |
| `SubagentStop` | when a sub-agent ends | sends it back to work | check the sub-agent's report has every field |
| `PreCompact` | before the context is summarised | blocks the compaction | save state, steer the summary |
| `SessionEnd` | when the session closes | | log cost, release locks |

### More events

Claude Code's hooks reference lists over thirty events. The others, grouped, with what exit 2 does where the reference says it blocks:

| Group | Events | Exit 2 |
|---|---|---|
| prompts | `UserPromptExpansion` | blocks the expansion |
| tools | `PermissionDenied`, `PostToolBatch` | `PostToolBatch` stops the loop before the next model call |
| sub-agents and tasks | `SubagentStart`, `TaskCreated`, `TaskCompleted`, `TeammateIdle` | rolls back creation, prevents completion, keeps a teammate working |
| context | `InstructionsLoaded`, `PostCompact` | |
| models | `PreModelSwitch`, `PostModelSwitch` | `PreModelSwitch` blocks the switch |
| files and folders | `CwdChanged`, `DirectoryAdded`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove` | for worktrees, any non-zero exit fails the operation |
| configuration | `Setup`, `ConfigChange` | `ConfigChange` blocks a change, except to policy settings |
| MCP | `Elicitation`, `ElicitationResult` | declines the request, blocks the response |
| turn end | `StopFailure` | |
| display | `MessageDisplay` | |

Read each event's entry in the reference for its input fields before writing a hook for it.

## Session events

### SessionStart

A `SessionStart` hook's plain-text output is added to the context, so it is a way to give every session facts that change daily without editing a context file. Its matcher picks the kind of start:

| Matcher | Fires on |
|---|---|
| `startup` | a new session |
| `resume` | `--resume`, `--continue` or `/resume` |
| `clear` | `/clear` |
| `compact` | after an automatic or manual compaction |

Example output:

```text
Git: branch feature/login, 2 files modified, 1 commit ahead of origin.
Open tasks assigned to this session: 2 (highest priority first).
Reminder: a release freeze is in effect until Friday.
```

#### Keep it short

- The hook runs on every start and resume, and its output costs tokens for the rest of the session.
- Each string it adds is capped at 10,000 characters.
- A few lines is the right size.
- A `compact` matcher is a way to put state back after a compaction.

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

Its plain-text output is added to the context, so keep it to a few lines and fire it only when its condition matches. Its default timeout is 30 seconds, shorter than most events, because the person is waiting.

## Tool events

### PreToolUse

The most used event. It sees the tool name and its full input before anything runs.

- Return `permissionDecision: "deny"` with a reason to refuse.
- Return `"allow"` to skip the prompt (a matching deny or ask rule still applies), `"ask"` to show one, or `"defer"` to leave it to the normal flow.
- Return `updatedInput` to change the arguments before the tool runs.
- Exit 2 with a message on standard error for a simple refusal.
- [Guard hooks]({{ '/sections/02-what-is-a-harness/guard-hooks.html' | relative_url }}) covers writing one in depth.

### PostToolUse

Runs after a tool succeeds, with both the input and the response. It cannot undo the action, but it can tell the model about a problem while the model still has the change in view.

- `decision: "block"` with a `reason` shows the reason next to the result.
- `additionalContext` adds text without the block wording.
- `updatedToolOutput` replaces what the model sees, for example to redact a secret from a command's output.

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

- A hook here can approve or refuse by code, through its `decision` object.
- Exit 2 does not refuse here, unlike `PreToolUse`.
- It suits rules too subtle for a [permission pattern]({{ '/sections/02-what-is-a-harness/permissions-and-settings.html' | relative_url }}), such as "allow `git push` only to branches this session created".

## Turn events

### Stop

Fires when the agent decides it is done. A hook that returns `"decision": "block"` with a `reason`, or exits 2, sends it back to work with that reason.

```json
{ "decision": "block", "reason": "2 tests fail: test_login_empty_email, test_login_trailing_space. Fix them before you finish." }
```

#### Worked example

A Stop hook runs the fast test suite. If any test fails, it blocks the stop with the failing test names. The agent cannot end its turn claiming success while the tests are red.

> A Stop hook that always blocks traps the agent in a loop. The input field `stop_hook_active` is true when a stop hook already sent the agent back; check it and let that stop through, with a note to the person.
{: .warning }

### SubagentStop

The Stop event for a sub-agent.

1. A sub-agent finishes and returns its report. The hook's input carries `agent_type` and `last_assistant_message`, the text of the sub-agent's final reply.
2. The hook checks that text for every required field: head commit, checks run and their results, anything unfinished.
3. If a field is missing, it blocks with the list of missing fields, and the sub-agent adds them.
4. The launcher collects a complete report.

My harness uses a check of this kind on sub-agent reports.

## Context events

### PreCompact

Fires before the harness summarises the context, for both a manual `/compact` and an automatic one. Detail in the conversation is about to be lost.

- Exit 2 blocks the compaction: use it to refuse until a resume note exists and is newer than the last commit.
- Plain text it prints becomes instructions for the summary, such as "keep the list of open decisions".
- Write the current task, open decisions and git state somewhere durable.
- `PostCompact` fires after, and a `SessionStart` hook with the `compact` matcher can put state back.
- See [Harness memory and context]({{ '/sections/02-what-is-a-harness/memory-and-context.html' | relative_url }}#compaction).

## Choosing the event

| You want to... | Use |
|---|---|
| stop an action before it happens | `PreToolUse` |
| check a result while it is fresh | `PostToolUse` |
| refuse to finish while something is wrong | `Stop` or `SubagentStop` |
| give every session today's facts | `SessionStart` |
| add context when a topic comes up | `UserPromptSubmit` |
| save state before detail is lost | `PreCompact` |
| put state back after a summary | `SessionStart` with `compact`, or `PostCompact` |
| change a tool's arguments or output | `PreToolUse` with `updatedInput`, `PostToolUse` with `updatedToolOutput` |
| record what a session did | `SessionEnd` |

## Sources

- Claude Code hooks reference, events, input fields and exit codes: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-09.
- The worked examples are from the author's own harness, described conceptually.

{% include page_nav.html %}
