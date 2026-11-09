---
title: "Memory files and lessons"
parent: "Context window and memory"
nav_order: 2
---

{% include page_nav.html %}

# Memory files and lessons

As of 2026-11-09.

Memory files carry what a session learned into the next session. This page covers what memory is for, how to write a lesson that transfers, how to lay out an index that stays inside its load limit, and how memory differs from context files and the task tracker. The [section overview]({{ '/sections/11-context-window/' | relative_url }}) is the short version.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Three stores, three jobs

| Store | Holds | Loads | Changes |
|---|---|---|---|
| [Context files]({{ '/sections/04-context-files/' | relative_url }}) | Standing rules for the project | Every session | Rarely, reviewed |
| Memory files | Lessons and current state | Every session (the index), on demand (lesson files) | Often, by the agent |
| Task tracker | What work exists and its status | When read | With every task |

A rule that should bind every session belongs in a context file. A lesson the agent learned the hard way, with its story, belongs in memory. A task belongs in the tracker. Mixing them makes each harder to keep short.

## Auto memory

Claude Code's auto memory is a memory file the agent writes and that loads again at session start and after compaction. Because it loads every time, its size is a cost every session pays.

## Writing a lesson

### The parts

| Part | Example |
|---|---|
| The rule, as a title | Count a message as delivered only when the target replies |
| What happened | A send timed out; I re-sent through the fallback; the target got the task twice |
| Why the rule follows | The timeout was about the sender, not the delivery |
| How to recognise the case | Any channel with a timeout or no delivery receipt |

### Why record the failure

A bare rule gets applied to the exact phrase it names. With the failure attached, the agent can see the shape of the problem and apply it to a new case that looks different on the surface: a test runner that times out, a deploy that reports "sent".

#### Example: rule alone versus rule with story

Rule alone: "Never re-send after a timeout." The agent later sees a push that timed out and assumes the rule is only about messages.

Rule with story: the agent sees that the general point is "a timeout is unknown, not failure", checks whether the push landed, and finds it did.

## The index

![A sequence diagram. Session 1 writes a lesson file and adds one line to the index. Days later session 2 loads the index at start, gets the pointer, reads the lesson file and gets the story behind the rule.]({{ '/assets/images/11-context-window/lesson-reaches-later-session.svg' | relative_url }}){: style="width:100%"}

*Figure: a lesson travels from one session to a later one through a short index. Made with archify.*

### One line per lesson, one file per lesson

The index loads every session, so it holds only a one-line pointer per lesson. The lesson's full text lives in its own file and is read when relevant.

```markdown
# Memory index

- [Delivered means replied](lesson-delivered-means-replied.md): a timeout is unknown
- [Test the merged result](lesson-test-the-merged-result.md): two green branches can break main
- [Earlier lessons](lessons-archive.md): older lines, read on demand
```

### Keep it under the load limit

> An index that grows past what loads drops its oldest lines without a warning. Keep indexes short and archive old entries.
{: .warning }

A memory index usually has a load limit, in lines or bytes. Past it, the rest is not loaded, and nothing tells the agent. The lessons at the end, often the newest or the oldest depending on where you append, silently stop applying.

#### Archiving

Move older lines to an archive index that is itself linked from the main index. The archive is read on demand, so its lessons are still reachable when a related problem comes up.

<details markdown="1">
<summary>A check that catches the silent drop</summary>

Count the index's lines or bytes on every change and fail when it passes a margin below the limit, say 80 percent of it. The check costs nothing and turns a silent loss into a visible task: archive some lines.

</details>

## State versus lessons

Memory can also hold current state: what a session is in the middle of. Keep that separate from lessons. State is overwritten often and is about now; lessons accumulate and are about always. One file per running session for state, and the lesson files for lessons, keeps either from crowding the other.

## The task tracker

A task tracker outside the session holds the real list of work. When a new request arrives mid-task, file it in the tracker and finish the current step, rather than keeping it in the conversation where the next compaction can lose it. The tracker also lets another session, or you, see the work without opening the session.

## Examples from my own harness

A memory index with one line per lesson and one file per lesson, each recording the failure behind it. Each running session has its own state file, so several sessions sharing one memory folder never overwrite each other's state. A task tracker is the record of all work; the task panel in the terminal only displays it.

To start: one notes file for state, and one lesson written the next time the agent makes the same mistake twice.

## Pitfalls and lessons

Lessons that only restate a rule do not transfer; record the failure. A memory file shared by several sessions needs one section or file per session. Reflexivity example: each lesson file records the failure behind it, so the agent can apply the general rule and not only the one phrase.

## Sources

- Memory and what reloads after compaction: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.

{% include page_nav.html %}
