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

- Claude Code's auto memory is a memory file the agent writes.
- It loads at session start and again after compaction.
- Because it loads every time, its size is a cost every session pays.
  - Keep it to an index and current state; put detail in files it points to.

## Writing a lesson

### The parts

| Part | Example |
|---|---|
| The rule, as a title | Count a message as delivered only when the target replies |
| What happened | A send timed out; I re-sent through the fallback; the target got the task twice |
| Why the rule follows | The timeout was about the sender, not the delivery |
| How to recognise the case | Any channel with a timeout or no delivery receipt |

### Why record the failure

- A bare rule gets applied to the exact phrase it names.
- With the failure attached, the agent can see the shape of the problem.
  - It can apply the rule to a case that looks different on the surface: a test runner that times out, a deploy that reports "sent".

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

## Where a fix belongs

A memory lesson or a new line in a context file guides the model, and the model applies it inconsistently: when it juggles several things, switches tasks, or works in a nearly full context window. Adding another detailed rule every time a model makes a mistake grows the context files without making the mistake impossible.

When a mistake matters or repeats, put the fix in the system, and keep the lesson as the second line of defence.

| Layer | Example for "re-sent a task after a timeout" | Enforces? |
|---|---|---|
| 1. Instructions: context files, skills, prompts, memory lessons | A lesson: "a timeout is unknown; ask before re-sending" | No: guides |
| 2. Workflows and tools that build the step in | The send tool tags each task with an id and asks the target before any re-send | Yes |
| 3. Hooks, scripts and linters at the moment of action | A hook refuses a second send of the same task id within an hour | Yes |
| 4. Validation and tests before a change lands | A test sends with a forced timeout and checks the target got the task once | Yes |

Layers 2 to 4 do the enforcing. The lesson file still helps: it records why the guard exists, so the next session understands it.

## State versus lessons

| | State | Lessons |
|---|---|---|
| About | What a session is in the middle of | What applies every time |
| Changes | Overwritten at every save | Accumulate |
| One file per | Running session | Lesson |
| Read | At resume | When the index points to one |

Keeping them apart stops either from crowding the other.

## The task tracker

- A task tracker outside the session holds the real list of work.
- When a new request arrives mid-task:
  1. File it in the tracker.
  2. Finish the current step.
  3. Pick the next task from the tracker.
- The tracker also lets another session, or you, see the work without opening the session.

## Examples from my own harness

- A memory index with one line per lesson and one file per lesson, each recording the failure behind it.
- One state file per running session, so several sessions sharing one memory folder never overwrite each other's state.
- A size check on the index, so it never grows past what loads.
- A task tracker as the record of all work; the task panel in the terminal only displays it.

To start: one notes file for state. When the agent repeats a mistake, look first for a check or a tool change that prevents it, and write the lesson to explain the check.

## Pitfalls and lessons

| Pitfall | What helps |
|---|---|
| Lessons that only restate a rule do not transfer | Record the failure behind it |
| A new rule for every mistake | Put the fix in a workflow, hook or test; keep the rule as the second layer |
| An index past its load limit drops lines silently | A size check, and an archive index |
| Several sessions writing one memory file | One section or file per session |

## Sources

- Memory and what reloads after compaction: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.

{% include page_nav.html %}
