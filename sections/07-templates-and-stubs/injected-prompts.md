---
title: "Injected prompts"
parent: "Templates, prompt stubs and injected prompts"
nav_order: 2
---

{% include page_nav.html %}

# Injected prompts

As of 2026-11-09. Topic page in [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }}). Slides: [talk stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

An injected prompt, on this site, is text the harness types into a session that is already running, at a moment the session cannot choose for itself. It wakes an idle session that still has a goal, reminds a session of its task after compaction, and sends the compact command before the window runs out. This page covers when a harness injects, why it has to type into the terminal, the four parts of an injection library, how to verify a submission, and how injection fails.

> Note: the security attack called prompt injection, where untrusted content smuggles instructions to a model, is a different thing. It is covered in [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

![A sequence across five participants. A background job finishes and tells the goal waker. The waker calls the injection library with the session name and text; the library clears the input, types and submits into the terminal pane. The pane still shows the text in the input box, so the library sends submit again, and the session starts a new turn to continue its goal. Later the session's context reaches 81 percent; the waker asks the library to save state and then send /compact with a focus line, and the pane reports that compaction finished.]({{ '/assets/images/07-templates-and-stubs/injection-wake.png' | relative_url }}){: style="width:100%"}

*Waking an idle session, then compacting it, through one library. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## When a harness injects

### The four triggers

| Trigger | What the session needs | Payload |
|---|---|---|
| Idle with a goal | To continue after a background job ends | "The job finished; continue the goal" |
| After compaction | The current task and rules the summary may have dropped | A short reminder naming the task and the rules |
| Context nearly full | To compact before it runs out of room | `/compact` with a focus line |
| Message from another session | To read and act on it | The message text, with its sender |

### Why the session cannot do it itself

Each trigger is something the harness can see and the session cannot.

- An idle session gets no turn until someone types, so it never notices that its background job ended.
- A session near a full window does not see the usage figure; that signal is outside its window.
- A message from another session arrives outside the session's own conversation.
- After compaction, the session cannot know which rule its own summary dropped.

## Why type into the terminal

### Commands sent as messages do not run

The obvious route is to send the text as a message.

- For a prompt, that works: the session reads it as a request.
- For a built-in command, it does not.
  - A command sent through cross-session messaging arrives as plain text.
  - A built-in command in a scheduled `/loop` fire is passed to Claude as text.
  - Claude Code's documentation says both.

```text
[message to session] /compact
[session] (receives the text "/compact" as a message; no compaction runs)
```

The transcript is illustrative: the session gets the characters, and the built-in command never executes.

### The terminal route

Text typed into the session's terminal input and submitted runs exactly as if a person typed it. Of the three routes in the table below, it is the only one that runs a built-in command at a time the harness chooses.

| Route | Prompt text | Built-in command |
|---|---|---|
| Cross-session message | Delivered | Arrives as text, does not run |
| Scheduled `/loop` fire | Delivered | Arrives as text, does not run |
| Typed into the terminal | Delivered | Runs |

## The library

A harness ends up injecting from several places. One shared library keeps them consistent.

| Caller | When it injects | Payload |
|---|---|---|
| Goal waker | A background job ended and the session is idle | "The job finished; continue the goal" |
| Context supervisor | Usage passes a threshold | Save state, then `/compact` with a focus line |
| Compaction path | Compaction finished | A reminder of the task and rules |
| Message relay | Another session sent a message | The message, with its sender |

Written separately, each caller would type and check differently, and a fix to one would miss the others.

### The four parts

#### Session target

Which session to type into: in a terminal multiplexer, the pane that session runs in. The target is resolved from the session's name at call time, never cached, because panes come and go.

#### Payload

The text to send: a prompt or a built-in command. The payload is plain text; the library does not interpret it.

#### Injector

Types the payload into the target and submits it. It handles the details that differ by terminal: clearing a half-typed input first, and sending the submit key separately from the text so a long payload does not get split.

#### Verify

Confirms the text was submitted and did not just sit in the input box. Without this step, an injection that typed but never submitted looks the same as one that worked.

One call through the library, in order (the figure at the top of this page shows it with a retry):

1. The caller passes a session name and a payload.
2. The library resolves the name to a pane, clears any half-typed input, and types the payload.
3. It sends the submit key as a separate keystroke.
4. It reads the input line. Empty, and a new turn started: it reports submitted.
5. Text still in the input: it sends submit once more, then reports failure with the pane contents if that does not work.

### One fix, every caller

Because every caller uses the same four parts, a change lands once.

1. A terminal update changes how the submit key has to be sent.
2. The fix goes into the injector.
3. The goal waker, the supervisor, the compaction path and the message relay all get it together.
4. One test of the injector covers all four callers.

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
[supervisor] inject reminder: "resume from saved state; current task is step 3"
```

The supervisor saves state first, so nothing in flight is lost, then compacts with a focus line naming the current task, then reminds.

## How injection fails

| Failure | Symptom | Guard |
|---|---|---|
| Typed, never submitted | Text sits in the input box; nothing happens | Verify step reads the input line |
| Wrong target | Text lands in another session | Resolve the target by name at call time |
| Typed over a person | A person's half-written prompt is mixed with the payload | Do not inject into a session a person is typing in |
| Injection loop | A waker re-wakes a session that is working | Inject only when idle, and rate-limit |
| Dry run arms a real one | A test of the injector types for real | Test against a scratch pane, never a live session |

> Warning: never type into another person's or another session's live pane by hand to "help". Use the library, with its target check and its verify step, or send a message.

<details markdown="1">
<summary>Injection and the person's keyboard focus</summary>

Typing into a terminal pane in a multiplexer does not move the person's cursor or window. Typing into a desktop window, by contrast, takes the keyboard from whoever is using it. A harness that injects should only use routes that leave the person's focus alone.

</details>

## Related pages

- [Context window]({{ '/sections/11-context-window/' | relative_url }}): when and why sessions compact.
- [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}): messaging between sessions.
- [Document and goal templates]({{ '/sections/07-templates-and-stubs/document-and-goal-templates.html' | relative_url }}#goal-templates): the goals an idle-session waker resumes.

## Sources

- Commands in cross-session messages arrive as text: <https://code.claude.com/docs/en/cross-session-messaging.md>, verified 2026-11-09.
- Built-in commands in scheduled `/loop` fires arrive as text: <https://code.claude.com/docs/en/scheduled-tasks.md>, verified 2026-11-09.
- The injection library, its four parts and its callers are from the author's own harness, described conceptually. The transcripts are illustrations written for this page.

{% include page_nav.html %}
