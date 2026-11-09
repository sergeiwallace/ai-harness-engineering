---
title: "Harness memory and context"
parent: "What is a harness?"
nav_order: 9
redirect_from:
  - /sections/02-ai-harnesses/memory-and-context.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Harness memory and context

- Part of [What is a harness?]({{ '/sections/02-what-is-a-harness/' | relative_url }}).
- The full section on this topic is [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}).

The context window is what the model can hold in one session.

- Memory is what carries from one session to the next.
- A task tracker is memory for work.
- Together they decide whether a long project gets smarter over time or repeats itself.

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

<!-- IMAGE NEEDED: a context window filling over one session, with the standing parts at the bottom (system prompt, tools, context files) and messages growing until compaction; search: LLM context window filling diagram, context window compaction, context engineering diagram -->

### What the size does to the work

- The window is finite, and use degrades before it is full: instructions are followed less consistently as more text competes for attention.
- Long tool results are the fastest way to fill it: a full test log, a large file read whole, a verbose command.
- Ways to keep it lean: read file ranges, filter command output, send long searches to a sub-agent.

### Measuring it

| Tool | Command | Shows |
|---|---|---|
| Claude Code | `/context`, `/context all` | a coloured grid of the split between system prompt, tools, MCP tools, memory files, skills and messages, with suggestions for heavy items |
| Codex | `/status` | context used and remaining |
| my harness | a small context-percent command | the percentage and a band: fine, plan a save, save now |

My harness has a rule that any claim about context must quote the current reading. Turn count and "it feels long" are not measurements.

## Compaction

### What happens

1. The window nears its limit, or you run `/compact`, optionally with instructions for what to keep.
2. `PreCompact` hooks run: one can block the compaction with exit 2, and plain text it prints steers the summary.
3. The harness replaces the conversation with a summary.
4. The project-root `CLAUDE.md` is re-read from disk. Folder `CLAUDE.md` files and path rules load again the next time a matching file is touched.
5. Detail that was only in the conversation is gone: exact error text, half-made decisions, which files were read, and any instruction you gave only in chat.

### Choosing the moment

Compact at a checkpoint you pick, before the harness forces it mid-step.

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

A `PreCompact` [hook]({{ '/sections/02-what-is-a-harness/hook-lifecycle-events.html' | relative_url }}#precompact) can check that state was saved before the summary runs.

## Memory files

![A sequence diagram of a lesson carried between sessions. Session 1 writes a lesson file and an index line to memory, saying what happened and how to spot it, and writes step 4 of 6 with notes to the tracker. Before compaction a PreCompact hook checks that a resume note is present. The next morning, session 2 starts with the memory index loaded (only its top part loads) and the ready and in-progress work from the tracker, then reads the lesson file and gets the check to run.]({{ '/assets/images/02-ai-harnesses/memory-across-sessions.png' | relative_url }})

*Memory holds lessons and the tracker holds work; both outlive the session. Our own diagram, made with archify.*

### Claude Code's auto memory

Claude Code keeps a memory folder per repository at `~/.claude/projects/<project>/memory/`.

| Fact | Detail |
|---|---|
| what loads at start | the first 200 lines or 25 KB of `MEMORY.md`, whichever comes first |
| topic files | not loaded at start; read with the normal file tools when needed |
| worktrees | every worktree and subfolder of one repository shares one memory folder |
| machines | machine-local; not shared across machines or cloud sessions |
| size | near a limit, Claude Code reminds the agent to shorten the index |
| who writes it | the agent, when it learns something; "remember that..." saves there, "add this to CLAUDE.md" goes to the context file |
| control | `/memory` to browse and toggle it; `autoMemoryEnabled: false` in settings to turn it off for a project |
| sub-agents | a profile's `memory` field gives a sub-agent memory of its own |

### The index

```markdown
# Memory index

One line each: past the read limit drops silently, tail first.

- [Check a merge in a scratch tree first](merge-in-scratch-first.md): a hand-resolved conflict once split a test from its asserts
- [A green push can hide an unrun check](green-push-unrun-check.md): read the verified line beside the exit code
- [Never type into a peer's terminal](never-type-into-peer.md): message it instead
```

Only the first 200 lines or 25 KB of the index load, and nothing warns the agent about the lines it did not see. My harness:

1. keeps one short line per lesson
2. moves older lines to an archive file the index links to
3. puts the most important lessons near the top
4. writes the read limit at the top of the index, so every editor sees it.

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
| memory | it is a lesson from experience, still being proven |
| a context file | it is short guidance for every session, decided and stable |
| a workflow, hook or test | it must hold every time |

A lesson that keeps proving itself moves into the system, following the [systems approach]({{ '/sections/02-what-is-a-harness/' | relative_url }}#a-systems-approach-to-harness-engineering): a skill step, a [hook]({{ '/sections/02-what-is-a-harness/hooks.html' | relative_url }}) or a test enforces it, and a short context-file line stays as a second layer.

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

## Pitfalls

| Pitfall | Result | Fix |
|---|---|---|
| an index past its read limit | the newest lessons silently stop loading | keep it short; archive old lines |
| two lessons that contradict | the agent follows either one | merge or delete the stale one |
| a secret or a host name in a lesson | it loads into every session | keep memory free of credentials; review it like code |
| a lesson that names a fixed bug | the agent works around a problem that no longer exists | prune lessons when the cause is fixed |
| state kept only in chat | lost at compaction | the tracker and a resume note |

## Worked example

A session works on a feature for three hours.

1. At a checkpoint it reads its context reading: high.
2. It writes the resume message and updates the issue's notes in the tracker.
3. It adds one memory line about a surprise in the build.
4. It compacts.
5. The summarised session reads the resume message, runs `bd ready`, and continues from step 4.
6. The next morning a new session reads the memory line at startup and avoids the build surprise.

## Sources

- [Claude Code memory, auto memory and compaction](https://code.claude.com/docs/en/memory.md), verified 2026-11-09.
- [Claude Code `/context` and `/compact`](https://code.claude.com/docs/en/commands.md), verified 2026-11-09.
- [`PreCompact` behaviour](https://code.claude.com/docs/en/hooks.md), verified 2026-11-09.
- [Beads](https://github.com/steveyegge/beads).
- The context reading, resume message, memory index rules and tracker rules are from the author's own harness, described conceptually.

{% include page_nav.html %}
