---
title: "Memory files and lessons"
parent: "Context window and memory"
nav_order: 2
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Memory files and lessons

Memory files carry what a session learned into the next session. This page covers what memory is for, Claude Code's auto memory, how to write a lesson that transfers, how to lay out an index that stays inside its load limit, where a fix for a repeated mistake belongs, and how memory differs from context files and the task tracker. The [section overview]({{ '/sections/11-context-window/' | relative_url }}) is the short version.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Three stores, three jobs

| Store | Holds | Loads | Changes |
|---|---|---|---|
| [Context files]({{ '/sections/04-context-files/' | relative_url }}) | Standing rules for the project | Every session, in full | Rarely, reviewed |
| Memory files | Lessons and current state | Every session (the index), on demand (lesson files) | Often, by the agent |
| Task tracker | What work exists and its status | When read | With every task |

- A rule that should bind every session belongs in a context file.
- A lesson the agent learned the hard way, with its story, belongs in memory.
- A task belongs in the tracker.
- Anything that must hold no matter what the model decides belongs in a hook, script or test; the [section below](#where-a-fix-belongs) covers this.

## Auto memory

Claude Code's auto memory is a set of notes Claude writes for itself from your corrections and preferences.

| Property | Value |
|---|---|
| Where | `~/.claude/projects/<project>/memory/`: a `MEMORY.md` index and one topic file per memory |
| Scope | One git repository, shared by all its worktrees and subdirectories, on one machine |
| Loaded at session start | The first 200 lines or 25KB of `MEMORY.md`, whichever comes first |
| Topic files | Read on demand, when Claude needs them |
| After compaction | Re-injected from disk |
| In sub-agents | Not loaded, except in a fork of the conversation |
| Near the limit | After Claude writes `MEMORY.md`, Claude Code measures it and reminds Claude to shorten it |

### What Claude saves

Each memory file records its kind in a `type` frontmatter field:

| `type` | Holds |
|---|---|
| `user` | Your role, expertise and working preferences |
| `feedback` | Corrections you gave and approaches you confirmed |
| `project` | Ongoing work, deadlines and decisions that the code and git history do not show |
| `reference` | Where to find things outside the project, such as a tracker or dashboard |

Claude skips what it can derive from the code, and what your `CLAUDE.md` already says.

### Controls

| Control | Effect |
|---|---|
| `/memory` | Opens memory and `CLAUDE.md` files; toggles auto memory |
| `"autoMemoryEnabled": false` | Turns it off, in user or project settings |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | Turns it off for a process |
| `"autoMemoryDirectory": "<path>"` | Stores it somewhere else |

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

1. The lesson file says only "Never re-send after a timeout."
2. Days later the agent's `git push` times out.
3. With the rule alone, it reads the rule as being about messages, and pushes again blind.
4. With the story attached ("a send timed out; the first one had arrived; the target got it twice"), it sees the general point: a timeout is unknown.
   - It checks whether the push landed, and finds it did.

## The index

![A sequence diagram. Session 1 writes a lesson file and adds one line to the index. Days later session 2 loads the index at start, gets the pointer, reads the lesson file and gets the story behind the rule.]({{ '/assets/images/11-context-window/lesson-reaches-later-session.svg' | relative_url }}){: style="width:100%"}

*Figure: a lesson travels from one session to a later one through a short index. Made with archify.*

### One line per lesson, one file per lesson

The index loads every session, so it holds only a one-line pointer per lesson. The lesson's full text lives in its own file and is read when relevant.

```markdown
# Memory index

- [Delivered means replied](feedback_delivered_means_replied.md): a timeout is unknown
- [Test the merged result](feedback_test_merged_result.md): two green branches can break main
- [Earlier lessons](lessons-archive.md): older lines, read on demand
```

A lesson file:

```markdown
---
type: feedback
---
# Count a message as delivered only when the target replies

What happened: a send timed out, I re-sent through the fallback, and the
target got the task twice. The first send had arrived.
Rule: treat a timeout as unknown. Ask the target before re-sending.
Applies to: any channel with a timeout or no delivery receipt.
Enforced by: the send tool refuses a second send of the same task id.
```

### Keep it under the load limit

| Index | Load limit | Warning when near it |
|---|---|---|
| Claude Code's `MEMORY.md` | 200 lines or 25KB | Yes: Claude Code reminds Claude after it writes the file |
| An index you build yourself and load from a context file or hook | Whatever loads it allows | Only if you add one |

#### Archiving

1. Move older lines to an archive index.
2. Link the archive from the main index.
3. The archive is read on demand, so its lessons are still reachable when a related problem comes up.

#### A size check

For any index, a check on every change turns a silent loss into a visible task. Fail at a margin below the limit, such as 80 percent:

```bash
#!/usr/bin/env bash
# fail when the memory index nears its load limit (200 lines or 25KB)
f="${1:-MEMORY.md}"
lines=$(wc -l < "$f"); bytes=$(wc -c < "$f")
if (( lines > 160 || bytes > 20480 )); then
  echo "$f is at $lines lines, $bytes bytes: archive old entries" >&2
  exit 1
fi
```

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
| An index past its load limit | One line per entry, an archive index, and a size check |
| Several sessions writing one memory file | One section or file per session |
| Expecting sub-agents to know the lessons | Auto memory is not loaded into sub-agents; put what they need in the brief |

## Sources

- Auto memory: location, load limit, note types, controls, sub-agents: [How Claude remembers your project (Claude Code docs)](https://code.claude.com/docs/en/memory.md), verified 2026-11-09.
- What reloads after compaction: [Explore the context window (Claude Code docs)](https://code.claude.com/docs/en/context-window.md), verified 2026-11-09.

{% include page_nav.html %}
