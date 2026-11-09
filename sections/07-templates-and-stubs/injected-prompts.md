---
title: "Injected prompts"
parent: "Templates, prompt stubs and injected prompts"
nav_order: 2
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# Injected prompts

An injected prompt, on this site, is text that reaches a session that is already running, at a moment the session cannot choose for itself: a wake-up when work finishes, a reminder after compaction, the compact command before the window fills. Claude Code now has native routes for most of these, and a harness should use them first. Typing into the session's terminal is the fallback for the cases no native route covers, chiefly running a built-in command such as `/compact` at a chosen moment.

> Note: the security attack called prompt injection, where untrusted content smuggles instructions to a model, is a different thing. It is covered in [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

![A sequence across five participants. A background job that runs outside the session finishes and tells the waker, labelled goal waker in the drawing. The waker calls the injection library with the session name and text; the library clears the input, types and submits into the terminal pane. The pane still shows the text in the input box, so the library sends submit again, and the session starts a new turn to continue its goal. Later the session's context reaches 81 percent; the waker asks the library to save state and then send /compact with a focus line, and the pane reports that compaction finished.]({{ '/assets/images/07-templates-and-stubs/injection-wake.png' | relative_url }}){: style="width:100%"}

*Figure 1. The terminal route: waking an idle session after a job that ran outside it, then compacting it. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a harness needs to send in

### Five needs and their routes

| Need | Native route in Claude Code | When the terminal route is still needed |
|---|---|---|
| Wake a session when its own background work ends | A background shell or sub-agent the session started wakes the idle session with its result when it finishes, with or without a goal | The work ran outside the session (a CI job, another machine) and the session did not start it |
| Pass news from another session | Cross-session messaging: a message to an idle session starts a new turn | The receiving session has no messaging (older version, bare mode) or refuses inbound messages |
| Push an external event, such as a CI result | Channels: an MCP server pushes events into the open session | Channels are unavailable: research preview, and not offered on Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry |
| Add context after compaction | A `SessionStart` hook with the `compact` matcher; its stdout becomes context | Rarely |
| Run a built-in command, such as `/compact`, at a chosen moment | None: messages and `/loop` fires deliver commands as text, and no hook is documented as running a slash command | Always |

### Choosing a route

1. Use the native route when one fits.
   - It is supported, versioned and documented, and it carries its own safeguards (inbound controls, rate limits, check-in caps).
2. Use a hook when the need is tied to a session event, such as compaction or session start.
   - Hooks are configuration, so the behaviour is the same in every session in their scope.
3. Use the terminal route only for what is left.
   - In practice that is built-in commands at a chosen time, and sessions or providers where the native routes are unavailable.

This follows the systems approach: the harness builds the behaviour into tools and hooks, and the hand-built terminal route carries only the cases nothing else can.

## The native routes

### Background work the session started

A background shell command or sub-agent started by the session wakes the idle session with its result when it finishes. This happens with or without a goal set.

1. The session starts a long command in the background, such as the full test suite, or a background sub-agent.
2. The session finishes its turn and goes idle.
3. The process finishes.
4. Claude Code starts a new turn with the result, and the session continues.

Watching is optional:

| Tool | Use it for |
|---|---|
| Claude Code's Monitor tool | Following a background process's progress before it finishes |
| A scheduled check, such as `/loop` or a cron job | Checking on work at intervals, including external work the session did not start |

#### Edge cases

| Case | What happens |
|---|---|
| The job runs outside the session, for example in CI, and the session did not start it | Claude Code does not see it end; a scheduled check, a channel or a harness waker has to tell the session |
| The process was started outside Claude Code, in another terminal | It is not associated with the session, so it does not wake it |

### Goal check-ins

Goal check-ins are a separate `/goal` feature for background work that runs long while a goal is active.

- If a sub-agent or background shell command is still running when a turn ends, the goal's evaluation is skipped for that turn.
- After 30 minutes of waiting on background work, a check-in is due: Claude lists the running tasks, reads their output, and fixes or stops stuck ones.
  - Later check-ins wait twice as long each time, up to four times the first interval: 1 hour, then every 2 hours.
  - In an interactive session, Claude Code starts at most three idle check-ins per goal between your prompts; after the third, they pause until you send another prompt.
  - In a non-interactive session (`claude -p`), check-ins arrive only at the end of a turn.
- `CLAUDE_CODE_GOAL_CHECKIN_MINUTES` changes the first interval; `0` turns check-ins and automatic retries off.

### Cross-session messages

- One session sends with the `SendMessage` tool; Claude finds the target with `ListAgents`, or you name it with `@session-name`.
- During an active turn, the receiving Claude reads the message between tool calls. When the session is idle, Claude Code starts a new turn with it.
- A command inside the message, such as `/compact`, arrives as plain text and never runs.
- `notify_when_idle` asks another session on the same machine for one notice when it next goes idle or exits; the subscription expires after 12 hours.
- The receiving session's `crossSessionInbound` setting decides what happens to arrivals:

| Value | Effect |
|---|---|
| `accept` | Delivered |
| `hold` | Shown as a notice, delivered only when an `accept` later applies |
| `refuse` | Dropped |

> Warning: with no `crossSessionInbound` value set, a session that bypasses permission prompts holds each incoming message for approval unless the sender also bypasses, and an unanswered approval dialog drops the message after five minutes by default (`dialogExpiry`). A harness that relies on messages to wake unattended sessions has to set `accept` explicitly.

### Hooks around compaction

Two hook events cover what a harness used to inject around compaction:

| Hook | Matcher | What its output does |
|---|---|---|
| `PreCompact` | `manual`, `auto` | Stdout becomes instructions for the compaction summary; exit code 2 blocks compaction |
| `SessionStart` | `compact` (also `startup`, `resume`, `clear`, `fork`) | Stdout, or `hookSpecificOutput.additionalContext`, is added as context |

#### Worked example: a reminder after every compaction

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          { "type": "command", "command": "cat .claude/after-compact.md" }
        ]
      }
    ]
  }
}
```

1. The session compacts, by hand or automatically.
2. Claude Code starts the post-compaction session and fires `SessionStart` with the `compact` source.
3. The hook prints the reminder file: the current task, where its state is saved, and the rules that matter.
4. Claude Code adds that text as context, so the session continues with the reminder in its window.

### Channels

- A channel is an MCP server that pushes events into the running session: CI results, chat messages, monitoring alerts.
- It is enabled per session with `claude --channels plugin:<name>@<marketplace>`.
- Every approved channel keeps a sender allowlist; other senders are dropped.
- Channels are a research preview: they need claude.ai or Console authentication, and they are not available on Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry.

## The terminal route

### Why it is still needed

| Route | Prompt text | Built-in command such as `/compact` |
|---|---|---|
| Cross-session message | Delivered; starts a turn when idle | Arrives as text, does not run |
| Scheduled `/loop` fire | Delivered | Passed to Claude as text, does not run |
| Hook output | Added as context | No documented way to run one |
| Typed into the terminal | Delivered | Runs, as if a person typed it |

```text
[message to session] /compact
[session] (receives the text "/compact"; no compaction runs)
```

The transcript is illustrative: the session gets the characters, and the built-in command never executes.

### The commands involved

In a terminal multiplexer such as tmux, typing into a session's pane takes three commands:

```bash
# type the text literally (-l), so tmux does not read words like "Enter" as key names
tmux send-keys -t "$pane" -l "/compact focus: docs rework, step 3 of 5"
# send the submit key as its own keystroke
tmux send-keys -t "$pane" Enter
# read the pane back to check the input line is empty and the session reacted
tmux capture-pane -p -t "$pane" | tail -n 5
```

<!-- IMAGE NEEDED: a terminal multiplexer window with several AI coding sessions in separate panes; search: tmux panes screenshot, tmux split window terminal -->

## The library

A harness that types into terminals from several places needs one shared library, so every caller types and checks the same way.

| Caller in my harness | When it uses the terminal route | Payload |
|---|---|---|
| External-job waker | A job the session did not start, such as a CI run, finished and the session is idle | "The CI run finished; read its result and continue" |
| Context supervisor | Usage passes a threshold | Save state, then `/compact` with a focus line |
| Compaction path | A compaction that should have fired did not | `/compact` |

Written separately, each caller would type and check differently, and a fix to one would miss the others.

### The four parts

#### Session target

- Which session to type into: in a terminal multiplexer, the pane that session runs in.
- Resolved from the session's name at call time, never cached, because panes come and go.

#### Payload

- The text to send: a prompt, or a built-in command such as `/compact`.
- Plain text; the library does not interpret it.

#### Injector

1. Clears any half-typed input in the pane.
2. Types the payload literally.
3. Sends the submit key as a separate keystroke.
   - Sending it with the text can submit before the text has arrived.

#### Verify

1. Reads the pane's input line after the submit.
2. Empty, and a new turn started: reports submitted.
3. Text still there: sends submit once more.
4. Still there: reports failure with the pane contents.

Without this step, an injection that typed but never submitted looks the same as one that worked.

### One fix, every caller

1. A terminal update changes how the submit key has to be sent.
2. The fix goes into the injector.
3. Every caller gets it together.
4. One test of the injector covers all callers.

## Verification in practice

### What to check

1. The input line is empty after submission.
2. A new turn started: the session shows activity it did not show before.
3. For `/compact`, the session reports that it compacted.

### Worked example: compacting before the window fills

```text
[supervisor] session "docs-rework": context 81%, threshold 80%
[supervisor] ask session to save state           -> submitted
[supervisor] session reports: state saved
[supervisor] inject "/compact focus: docs rework, step 3 of 5" -> submitted
[supervisor] verify: session reports compaction finished
[hook]       SessionStart(compact): reminder file added as context
```

1. The supervisor asks the session to save its state, so nothing in flight is lost.
2. It types `/compact` with a focus line naming the current task, the one step only the terminal route can do.
3. It verifies that compaction ran.
4. The `SessionStart` hook adds the reminder, so the supervisor does not have to type one.

## How injection fails

| Failure | Symptom | Guard |
|---|---|---|
| Typed, never submitted | Text sits in the input box; nothing happens | Verify step reads the input line |
| Wrong target | Text lands in another session | Resolve the target by name at call time |
| Typed over a person | A person's half-written prompt is mixed with the payload | Do not inject into a session a person is typing in |
| Injection loop | A waker re-wakes a session that is working | Inject only when idle, and rate-limit |
| Dry run arms a real one | A test of the injector types for real | Test against a scratch pane, never a live session |
| Native route already did it | The session gets the same wake-up twice | Skip the terminal route for background work the session started, and when a message covers the case |

> Warning: never type into another person's or another session's live pane by hand to "help". Use the library, with its target check and its verify step, or send a message.

<details markdown="1">
<summary>Injection and the person's keyboard focus</summary>

- Typing into a terminal multiplexer pane does not move the person's cursor or window.
- Typing into a desktop window takes the keyboard from whoever is using it.
- A harness that injects should only use routes that leave the person's focus alone.

</details>

## Related pages

- [Context window]({{ '/sections/11-context-window/' | relative_url }}): when and why sessions compact.
- [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}): messaging between sessions.
- [Document and goal templates]({{ '/sections/07-templates-and-stubs/document-and-goal-templates.html' | relative_url }}#goal-templates): writing the goal a session works toward.
- [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}): hook events and configuration.

## Sources

- Goal check-ins and `CLAUDE_CODE_GOAL_CHECKIN_MINUTES`: [Claude Code docs: Keep Claude working toward a goal](https://code.claude.com/docs/en/goal.md), verified 2026-11-09.
- A background shell or sub-agent started by the session waking it on completion, with or without a goal: Claude Code behaviour as used by the author.
- Cross-session messaging, delivery, `crossSessionInbound`, `notify_when_idle`, commands arriving as text: [Claude Code docs: Message your other Claude Code sessions](https://code.claude.com/docs/en/cross-session-messaging.md), verified 2026-11-09.
- `SessionStart` and `PreCompact` matchers and output: [Claude Code docs: hooks reference](https://code.claude.com/docs/en/hooks.md), verified 2026-11-09. The hooks reference documents no way for a hook to run a slash command; that is an absence in the docs as read, not a stated limit.
- Channels: [Claude Code docs: Push events into a running session with channels](https://code.claude.com/docs/en/channels.md), verified 2026-11-09.
- Built-in commands in scheduled `/loop` fires arrive as text: [Claude Code docs: scheduled tasks and /loop](https://code.claude.com/docs/en/scheduled-tasks.md), verified 2026-11-09.
- `tmux send-keys -l` and `capture-pane -p`: the tmux manual page.
- The injection library, its four parts and its callers are from the author's own harness, described conceptually. The transcripts are illustrations written for this page.

{% include page_nav.html %}
