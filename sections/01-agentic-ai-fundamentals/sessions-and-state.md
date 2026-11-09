---
title: "Sessions and state"
parent: "Agentic AI fundamentals"
nav_order: 4
---

{% include page_nav.html %}

# Sessions and state

As of 2026-11-09.

A model keeps nothing between calls, so everything an agent "remembers" is something the client sends again or something written to a file. A session is the client's record of one conversation. This page explains what a session holds, how to resume, continue and fork one, how to hand work from one session to the next, and which kind of state belongs where. The [Agentic AI fundamentals overview]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}#sessions-and-state) is the short version, and [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}) covers memory systems in more depth.

> The model keeps nothing between sessions. Work carries over through files the next session reads: context files, plan files, memory files and git history.
{: style="border-left: 4px solid #0f766e; background: #f0fdfa; padding: 0.75rem 1rem;"}

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a session is

### The transcript

Claude Code records each session as a transcript on disk: every message, tool call and tool result, in order.

| The transcript lets you | How |
|---|---|
| scroll back through the session | the terminal history, or the transcript file |
| resume it later | `claude --continue` or `claude --resume` |
| audit what the agent did | read the tool calls and their results |

After compaction, the live context holds a summary while the full record stays on disk. The agent works from the summary; only you, or a tool that reads the file, can see the rest.

### What a session holds and what it does not

| Held in the session | Not held anywhere unless written down |
|---|---|
| the conversation and every tool result | decisions you made in your head |
| the current working directory and environment | why a design was chosen, unless it was said |
| the task list the agent keeps for itself | what a later session should do next |
| the model and settings in use | anything a compaction summary dropped |

### Sessions and the model

```text
turn 1:  client sends [system, files, task]                     -> model replies
turn 2:  client sends [system, files, task, reply 1, result 1]  -> model replies
turn 3:  client sends [system, files, task, ... , result 2]     -> model replies
```

The model sees a longer list each time; it has no memory of turn 1 other than turn 1 being in the list. Close the session and the list is gone from the model's point of view.

## Resuming, continuing and forking

### The commands

| Goal | Claude Code | What you get |
|---|---|---|
| Pick up the most recent session in this directory | `claude --continue` | the same transcript, the conversation carries on |
| Pick any earlier session | `claude --resume`, or `/resume` inside a session | a list of sessions to choose from |
| Try another approach without losing this one | fork the session (resume with the fork option, or `/fork`) | a copy that diverges from here |
| Start clean | a new `claude`, or `/clear` | an empty conversation; context files still load |

Exact flags change between versions; `claude --help` and the documentation list the current ones.

### When to resume and when to start fresh

#### Resume

The task is the same, the transcript is not too long, and the details in it (errors seen, files read) are still useful.

#### Start fresh

The task is different, the old session is near its window limit or has been compacted several times, or it went down a wrong path you do not want the model to keep seeing.

> A resumed session still contains its earlier mistakes. If the last session went down a wrong path, start a fresh session with a handoff note.
{: style="border-left: 4px solid #b91c1c; background: #fef2f2; padding: 0.75rem 1rem;"}

## Handoff notes

### Why write one

A handoff note is a short file the next session reads first. It carries work across sessions more reliably than the transcript:

- it survives compaction, closing the terminal and switching machines;
- it holds only what the next session needs, so it costs little context;
- a person can read and correct it too.

![Sequence diagram: in the afternoon you ask session 1 to migrate the orders schema; context files load; it commits tables A and B; you ask it to wrap up and it writes the plan file with done, next and watch out. The next morning you ask session 2 to continue; it loads the context files and the plan file, so it knows the next table and the nullable column; it runs make migrate for orders and updates the plan file]({{ '/assets/images/01-agentic-ai-fundamentals/session-handoff.svg' | relative_url }}){: style="width: 100%" }
*Figure: two sessions on one task. The only link between them is what session 1 wrote to disk.*

### What goes in it

```markdown
# Migration: orders schema v3

## Done
- Tables `customers` and `addresses` migrated; tests green.

## Next
- Migrate `orders`. Run `make migrate TABLE=orders` then `make test-db`.

## Watch out
- `orders.customer_id` is nullable in production data (about 2% of rows).
  The migration must keep the null values.

## Decisions
- Kept the old column for one release so rollback stays possible.
```

Four headings cover almost every case: what is done, what is next (with the exact command), what to watch out for, and decisions with their reasons.

### Worked example: resuming the next morning

An agent spends an afternoon on a migration and stops halfway, after writing the note above.

1. The next morning you start a fresh session and say "continue the migration".
2. The session loads the context files and reads the plan file first.
3. From the note it knows:
   - which table is next, `orders`;
   - the exact command, `make migrate TABLE=orders`;
   - the trap, about 2% of rows with a null `customer_id`.
4. It runs the migration, keeps the null values, and updates the plan file.

Without the note, step 3 becomes reading code and git history, and the null values are found when the migration fails.

### Making it automatic

<details markdown="1">
<summary>Three ways to make sure the note gets written</summary>

1. An instruction in the project's context file: "before stopping, or when asked to wrap up, update the plan file with done, next, watch out and decisions".
2. A skill or command that writes the note in a fixed format, which you or the agent can run before a planned stop or a compaction.
3. A `Stop` or pre-compaction hook that checks the plan file was updated in this session and reminds the agent if not.

The first works most of the time; the third works every time. [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers the trade-off.
</details>

## Where each kind of state belongs

### The map

| Kind of state | Lifetime | Best home | Loaded how |
|---|---|---|---|
| Project rules and conventions | months | context file (`CLAUDE.md`, `AGENTS.md`) | automatically at every start |
| Personal preferences | months | your user-level context file | automatically, in every project |
| Lessons learned from mistakes | weeks to months | a memory file, or promoted into a rule | at start, or on demand |
| Current task progress | days | a plan file or task tracker | read first by the next session |
| What changed and why | permanent | git commits and their messages | `git log` when needed |
| The detailed record of one run | as long as you keep it | the session transcript | resume, or read for an audit |

### Worked example: one fact, three places

A session discovers that the test database must be reset before the integration suite, or tests fail at random.

1. For the task in hand, it goes in the plan file under "watch out", so the next session on this task knows.
2. Because it is true for every task in the project, the fix goes into the system:
   - the `test-integration` target resets the database itself, so no session can run the suite against a dirty database;
   - a CI job runs the same target, so a change that breaks the reset fails before it lands.
3. A one-line note in the project's context file stays as a second layer: "the integration target resets the test database; do not run the suite by hand".

Writing only a new context-file rule ("run `make reset-test-db` before `make test-integration`") would rely on every session remembering it. Building the step into the target removes the need to remember. The introduction describes this [systems approach]({{ '/' | relative_url }}#a-systems-approach).

## Pitfalls

- Relying on the transcript as memory. Compaction and fresh sessions lose it; write decisions, next steps and traps to files.
- Resuming a confused session. A clean start with a good note beats carrying the confusion.
- Notes without commands. "Continue the migration" is weak; "run `make migrate TABLE=orders`" is a handoff.
- State in two places that disagree. If the plan file and the tracker say different things, the next session picks one at random.

## Sources

- Resuming and continuing sessions: <https://code.claude.com/docs/en/cli-reference.md>, not yet verified against the primary source for this page's date.
- Memory and context files: <https://code.claude.com/docs/en/memory.md>, not yet verified against the primary source for this page's date.
- The handoff note, transcript sketch and examples are illustrations written for this page.

{% include page_nav.html %}
