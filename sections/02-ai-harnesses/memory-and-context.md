---
title: "Harness memory and context"
parent: "AI harnesses"
nav_order: 9
---

{% include page_nav.html %}

# Harness memory and context

As of 2026-11-09. Part of [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}). The full section on this topic is [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}).

The context window is what the model can hold in one session. Memory is what carries from one session to the next. A task tracker is memory for work. Together they decide whether a long project gets smarter over time or repeats itself.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The context window

### What fills it

| Part | Loaded | Changes during the session |
|---|---|---|
| System prompt | at start | no |
| Tool and MCP descriptions | at start | when servers connect |
| Context files | at start, path rules on demand | path rules come and go |
| Skill summaries | at start | full bodies arrive on use |
| Memory index | at start | when the agent writes memory |
| Messages and tool results | every turn | grows until compaction |

### Why it matters

- The window is finite, and use degrades before it is full: instructions are followed less consistently as more text competes for attention.
- Long tool results are the fastest way to fill it: a full test log, a large file read whole, a verbose command.
- Ways to keep it lean: read file ranges, filter command output, send long searches to a sub-agent.

### Measuring it

| Tool | Command | Shows |
|---|---|---|
| Claude Code | `/context` | the split between system prompt, tools, memory, skills and messages |
| Codex | `/status` | context used and remaining |
| my harness | a small context-percent command | the percentage and a band: fine, plan a save, save now |

My harness has a rule that any claim about context must quote the current reading. Turn count and "it feels long" are not measurements.

## Compaction

### What happens

1. The window nears its limit, or you run `/compact`.
2. The harness replaces the conversation with a summary.
3. Context files and the memory index reload.
4. Detail that was only in the conversation is gone: exact error text, half-made decisions, which files were read.

### Choosing the moment

Compact at a checkpoint you pick, not when the harness forces it mid-step.

- Good moments: after a commit, after a sub-agent's work is collected, between tasks.
- Bad moments: halfway through a multi-file edit, while a decision is open.
- Before compacting, write down the current task and step, open decisions, git state and what is next.

### A resume message

```text
Resume: current task is the export-format change, step 4 of 6 (tests pass, docs not written).
Plan: docs/plans/export-format.md. Open decision: whether to keep the old flag.
Git: branch feature/export, 3 commits ahead, clean tree.
Next: write the docs page, then run the full suite and open the PR.
Rules: never push to main; ask before adding a dependency.
```

A `PreCompact` [hook]({{ '/sections/02-ai-harnesses/hook-lifecycle-events.html' | relative_url }}#precompact) can check that state was saved before the summary runs.

## Memory files

![A sequence diagram of a lesson carried between sessions. Session 1 writes a lesson file and an index line to memory, saying what happened and how to spot it, and writes step 4 of 6 with notes to the tracker. Before compaction a PreCompact hook checks that a resume note is present. The next morning, session 2 starts with the memory index loaded (only its top part loads) and the ready and in-progress work from the tracker, then reads the lesson file and gets the check to run.]({{ '/assets/images/02-ai-harnesses/memory-across-sessions.png' | relative_url }})

*Memory holds lessons and the tracker holds work; both outlive the session. Our own diagram, made with archify.*

### Claude Code's auto memory

Claude Code keeps a memory folder per project.

- A `MEMORY.md` index loads at the start of each session.
- Individual memory files are linked from the index and read when relevant.
- The agent writes to it when it learns something worth keeping.
- You can open and edit it with `/memory`.

### The index

```markdown
# Memory index

One line each: past the read limit drops silently, tail first.

- [Check a merge in a scratch tree first](merge-in-scratch-first.md): a hand-resolved conflict once split a test from its asserts
- [A green push can hide an unrun check](green-push-unrun-check.md): read the verified line, not the exit code
- [Never type into a peer's terminal](never-type-into-peer.md): message it instead
```

Only the first part of the index loads. My harness keeps one short line per lesson, moves older lines to an archive file the index links to, and puts the most important lessons near the top.

### A lesson file

```markdown
---
name: A green push can hide an unrun check
type: lesson
---

The push tool exits 0 when its second-tier checks are skipped for time.
Read the VERIFIED line in its output. If it does not list tier 2, run tier 2 by hand
before calling the push verified.
```

A good lesson says what happened, how to recognise it next time, and what to do. It is a rule with its evidence attached.

### Memory and context files

| Put it in | When |
|---|---|
| a context file | it is a rule for every session, decided and stable |
| memory | it is a lesson from experience, still being proven |
| a hook | it must hold every time |

A lesson that keeps proving itself graduates into a context file or a [hook]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}).

## Task tracking

### What a tracker gives an agent

| Need | Tracker answer |
|---|---|
| What should I work on? | ready issues by priority, with dependencies resolved |
| What was I doing? | issues in progress, with notes |
| Who owns this? | an assignee, claimed atomically |
| Why was this done? | history and links between issues |

### Beads

I use Beads, an open-source issue tracker built for coding agents. Issues live with the repository, the agent reads and writes them from the command line, and dependencies decide which issues are ready.

```text
$ bd ready
1. [P1] Export format: write docs page
2. [P2] Remove the old export flag (blocked by 1)
$ bd update <id> --claim
```

### Rules that make it work

1. The tracker is the only task record. A task mentioned only in chat is lost at compaction.
2. A new request that arrives mid-task is filed as an issue, and the current step is finished first.
3. Every sub-agent launch gets its own child issue, closed when its work is collected.

## Worked example

A session works on a feature for three hours.

1. At a checkpoint it reads its context reading: high.
2. It writes the resume message and updates the issue's notes in the tracker.
3. It adds one memory line about a surprise in the build.
4. It compacts.
5. The summarised session reads the resume message, runs `bd ready`, and continues from step 4.
6. The next morning a new session reads the memory line at startup and avoids the build surprise.

## Sources

- Claude Code memory: <https://code.claude.com/docs/en/memory.md>, verified 2026-11-08.
- Beads: <https://github.com/steveyegge/beads>.
- The context reading, resume message, memory index rules and tracker rules are from the author's own harness, described conceptually.

{% include page_nav.html %}
