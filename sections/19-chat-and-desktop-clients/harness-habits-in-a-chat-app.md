---
title: "Harness habits in a chat app"
parent: "Chat, desktop and web clients"
nav_order: 1
redirect_from:
  - /sections/19-chat-and-desktop-clients/detail.html
  - /sections/19-chat-and-desktop-clients/advanced.html
---

{% include page_nav.html %}

# Harness habits in a chat app

As of 2026-11-09. Topic page in [Chat, desktop and web clients]({{ '/sections/19-chat-and-desktop-clients/' | relative_url }}).

A chat app has no hooks, scripts or tests between the model and you. It has a place for standing instructions, a place for files, and your own review. This page takes four habits from the terminal harness and shows how each one works when you are the only enforcement.

![Sequence diagram of carrying the durable-notes habit into a chat project. In session 1 you add an instruction to end each chat with a notes block, work on the task, and get an answer plus a notes block; between sessions you paste the notes into the project's notes file; in session 2 you ask the model to continue from the notes, it reads the file and picks up at the right step, and you check the answer against the original ask.]({{ '/assets/images/19-chat-and-desktop-clients/chat-project-notes-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. The durable-notes habit across two chat sessions, with you doing the step a hook would do in a terminal harness. Diagram made with archify.*

## The four habits, side by side

| Habit | In a terminal harness | In a chat app |
|---|---|---|
| Durable notes | The agent writes notes to a file; a hook can check they exist | The model writes a notes block; you save it to the project |
| Keep the original ask | A template carries the request verbatim into every brief | You paste the request at the top of each new chat |
| Check before relying | Tests and gates check the result | You check the result against a written checklist |
| Decide what needs approval | Permission rules ask before an action | The model cannot act; you decide what to do with its output |

## 1. Durable notes

### Why it matters

A chat conversation ends, or grows too long, and the next one starts from nothing. Durable notes carry the state across: what was decided, what is still open, and what comes next. In a terminal harness this is [memory and the context window]({{ '/sections/11-context-window/' | relative_url }}). In a chat app it is a notes file in the project.

### Worked example

The project instruction:

```text
End every reply with a "Notes" block in this format:
Decided: ...
Open: ...
Next: ...
```

A reply ends like this:

```text
Notes
Decided: use the quarterly figures, not the monthly ones; drop the appendix.
Open: whether the second chart needs the 2025 baseline.
Next: draft section 3 from the outline in the notes file.
```

You paste the block into a `notes.md` file in the project. The next chat starts with "Continue from the notes file," and the model picks up at "Next".

#### What you lose without a hook

Nothing checks that the notes were written or saved. If a reply has no notes block, ask for one before you move on.

## 2. Keep the original ask

### Why it matters

Over a long conversation, the request drifts. Each answer reshapes the next question, and the final result can drift away from what you first asked for. A terminal harness keeps the request word for word and checks the result against it. In a chat app, you do the same by hand.

### Worked example

```text
Original request (verbatim, do not rephrase):
"A one-page summary of the Q3 survey for the leadership meeting: three findings,
one chart, and the decision we need from them."

Before you answer, list which parts of the original request your draft covers and which it does not.
```

Paste the first block at the top of each new chat about the task. The second line makes the model check its own coverage, and you check the list.

## 3. Check before relying

### Why it matters

A chat model's answer sounds equally confident when it is right and when it is wrong. In a terminal harness, [tests]({{ '/sections/12-testing/' | relative_url }}) and [review]({{ '/sections/13-code-review/' | relative_url }}) check the result. In a chat app, the check is a written checklist that you run.

### Worked example

<details markdown="1">
<summary>A checklist for a factual summary</summary>

1. Every number in the summary appears in the source file. Find each one.
2. Every claim names where it came from.
3. Nothing in the summary contradicts the source's own conclusion.
4. The summary answers the original request, item by item.

</details>

You can ask the model to run the checklist on its own answer, which catches some problems.

> Warning: a model checking its own answer often repeats the mistake it made. For anything that matters, check in a fresh chat, with a different model, or by reading it yourself.

## 4. Decide what needs your approval

### Why it matters

In a terminal harness, [human gates]({{ '/sections/14-human-gates/' | relative_url }}) stop the agent before outward or destructive actions. A chat model cannot act on your behalf, so the gate moves: it sits between the model's output and what you do with it. Decide in advance which outputs you will send, publish or act on without a second look, and which you will always check.

| Output | Use directly? |
|---|---|
| A first draft for your own editing | Yes |
| A message to a colleague | Read it first |
| Numbers that go into a decision | Check each one against the source |
| Anything published | Read it and check the facts |

## Examples from my own harness

My harness runs in the terminal clients, where each of these habits is a rule or a mechanism: durable notes in memory files, the original request kept verbatim in every brief, checks before any claim of done, and approval rules for outward actions. The chat versions on this page are those same habits with you doing the enforcement.

To start: add the notes-block instruction to one chat project and keep a notes file for one task.

## Sources

- Surfaces and what they share: <https://code.claude.com/docs/en/platforms.md>, verified 2026-11-08.
- Project instructions and files as the chat app's context: unverified; check the claude.ai help pages.

{% include page_nav.html %}
