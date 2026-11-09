---
title: "Harness habits in a chat app"
parent: "Chat, desktop and web clients"
nav_order: 1
redirect_from:
  - /sections/19-chat-and-desktop-clients/detail.html
  - /sections/19-chat-and-desktop-clients/advanced.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Harness habits in a chat app

A chat app has no hooks, scripts or tests between the model and you. It has a place for standing instructions, a place for files, and your own review. This page takes five habits from the terminal harness and shows how each one works when you are the only enforcement.

![Sequence diagram of carrying the durable-notes habit into a chat project. In session 1 you add an instruction to end each chat with a notes block, work on the task, and get an answer plus a notes block; between sessions you paste the notes into the project's notes file; in session 2 you ask the model to continue from the notes, it reads the file and picks up at the right step, and you check the answer against the original ask.]({{ '/assets/images/19-chat-and-desktop-clients/chat-project-notes-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. The durable-notes habit across two chat sessions, with you doing the step a hook would do in a terminal harness. Diagram made with archify.*

## The five habits, side by side

| Habit | In a terminal harness | In a chat app |
|---|---|---|
| Durable notes | The agent writes notes to a file; a hook can check they exist | The model writes a notes block; you save it to the project |
| Keep the original ask | A template carries the request verbatim into every brief | You paste the request at the top of each new chat |
| Check before relying | Tests and gates check the result | You check the result against a written checklist |
| Decide what needs approval | Permission rules ask before an action | The model cannot act; you decide what to do with its output |
| One task per session | Each task gets its own session or sub-agent with its own context | You start a new chat for each task |

## 1. Durable notes

### Why it matters

A chat conversation ends, or grows too long, and the next one starts from nothing. Durable notes carry the state across:

- What was decided.
- What is still open.
- What comes next.

| | Terminal harness | Chat app |
|---|---|---|
| Where the notes live | Memory files; see [the context window]({{ '/sections/11-context-window/' | relative_url }}) | A notes file in the project |
| Who writes them | The agent, often prompted by a workflow step | The model, prompted by the project instruction |
| Who saves them | The agent | You |

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

Over a long conversation, the request drifts. Each answer reshapes the next question, and the final result can move away from what you first asked for.

1. A terminal harness carries the request word for word.
   - A template copies it into every brief for a sub-agent.
   - A final check compares the result with it before "done".
2. In a chat app, you do both steps by hand.
   - Paste the request at the top of each new chat.
   - Compare the final answer with it yourself.

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

A chat model's answer sounds equally confident when it is right and when it is wrong.

| | Terminal harness | Chat app |
|---|---|---|
| What checks the result | [Tests]({{ '/sections/12-testing/' | relative_url }}), validators and [review]({{ '/sections/13-code-review/' | relative_url }}) | A written checklist |
| Who runs the check | A program or a reviewer agent | You |
| When it runs | Before the result lands, every time | When you remember to |

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

## 5. One task per chat

### Why it matters

Instructions are applied less consistently when a model juggles several things at once or switches between tasks, and when its context fills up. A coding harness avoids this by design: each task gets its own session or sub-agent. In a chat app, the design choice is yours.

| Situation | What to do |
|---|---|
| A new, unrelated task | Start a new chat in the same project, so the project instructions still load |
| The same task, but the chat has grown long | Ask for a notes block, start a new chat, and continue from the notes |
| A side question in the middle of a task | Ask it in a separate chat, and bring back only the answer |

### Worked example

```text
Chat 1 (drafting the survey summary), after many turns:
> Before we stop: write the Notes block.

Chat 2 (new chat, same project):
> Continue the survey summary from the notes file. Original request:
> "A one-page summary of the Q3 survey for the leadership meeting: three findings,
> one chart, and the decision we need from them."
```

The second chat starts with the project instructions, the notes and the original request, and none of the earlier back-and-forth.

## Examples from my own harness

My harness runs in the terminal clients, where each habit is built into the system:

| Habit | How the terminal harness enforces it |
|---|---|
| Durable notes | Memory files, written as a workflow step |
| The original request | Carried verbatim in every brief by a template |
| Check before relying | Checks that run before any claim of done |
| Approval | Approval rules for outward actions |
| One task per session | A separate session or sub-agent for each task, each with its own brief |

The chat versions on this page are the same habits with you doing the enforcement.

To start: add the notes-block instruction to one chat project and keep a notes file for one task.

## Sources

- Surfaces and what they share: [Claude Code docs: Platforms](https://code.claude.com/docs/en/platforms.md), verified 2026-11-08.
- Project instructions and files as the chat app's context: unverified; check the claude.ai help pages.

{% include page_nav.html %}
