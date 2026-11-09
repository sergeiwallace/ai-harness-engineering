---
title: "Compaction and saving state"
parent: "Context window and memory"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/11-context-window/detail.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Compaction and saving state

Compaction replaces a session's history with a summary so the session can keep going. It is necessary on any long task and it loses detail every time. This page covers what survives, when to compact, the three ways to shrink a conversation, how to steer the summary, how to write state so nothing important depends on the summary, how to resume automatically, and the edge cases. The [section overview]({{ '/sections/11-context-window/' | relative_url }}) is the short version. Sub-topic pages cover [a context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }}) that automates the routine, and [context in Claude Code and Codex]({{ '/sections/11-context-window/claude-code-and-codex-context.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What compaction does

![A sequence diagram: the context hook reports 58% used; the session writes a state note and updates the tracker; it runs /compact with a focus and gets a summary back; it reads its state note back and resumes.]({{ '/assets/images/11-context-window/save-state-and-compact.svg' | relative_url }}){: style="width:100%"}

*Figure: the order that keeps a compaction safe. Made with archify.*

1. The window nears its limit, or you run `/compact`.
2. A model summarises the conversation into a structured summary. Since v2.1.198 the summary request uses the session's extended-thinking setting.
3. The conversation is replaced by the summary.
4. Claude Code re-injects some content from disk and re-reads some files (next table).
5. The session continues from there.

### What survives

<!-- IMAGE NEEDED: a context window filling over one session, from startup content through file reads to a compaction, showing what reloads; search: claude code context window timeline, context window filling diagram, LLM context compaction diagram -->

| Content | After compaction |
|---|---|
| System prompt and output style | Still apply |
| Project-root `CLAUDE.md` and rules without `paths:` | Re-injected from disk |
| Auto memory | Re-injected from disk |
| Skill descriptions | Rebuilt and re-injected (v2.1.304 or later) |
| Git status snapshot | A fresh one is read, if the session started with one |
| A plan written in plan mode | Re-injected from disk |
| Rules with `paths:`, nested `CLAUDE.md` | Reloaded only when a matching file is touched again |
| Files Claude read or edited | Up to five re-read, most recently modified first; a file over 5,000 tokens comes back as a path reference only |
| Invoked skill bodies | Re-injected, capped at 5,000 tokens per skill and 25,000 in total; oldest dropped first; truncation keeps the start of the file |
| Background commands and sub-agents | Keep running; Claude is reminded which ones are still running |
| Context that hooks added earlier | Summarised with the rest |
| `SessionStart` hooks matching `compact` | Run, and their output is added |
| Files on disk | Untouched |

### What is lost

| Lost or blurred | Why it matters |
|---|---|
| Exact error messages and line numbers | The next step often depends on them |
| Decisions and the reason for each | The session may reopen a settled question |
| What was already tried | It may try the same failed fix again |
| Rules you gave in conversation | Unless they were written to a context file |
| Path-scoped rules loaded mid-session | They load again only when matching files are touched |
| Files beyond the five most recent | They have to be read again |

## When to compact

### By fill level

As the context window fills, an agent follows instructions less reliably.

| Setting or practice | Value |
|---|---|
| My own compaction point | About 50 to 60 percent of the window |
| Other practitioners' reports | A sweet spot below about 300k tokens (unverified practitioner reports) |
| Claude Code's automatic compaction | Near the limit; the exact point depends on the model and configuration |
| Moving the automatic threshold | `/autocompact` with a token count, such as `/autocompact 500k` |

### By task boundary

The best moment is between tasks: the last task is committed, its state written, and the next has not started. A compaction in the middle of a debugging session loses the most.

#### Example: choosing the moment

1. At 52 percent the session is halfway through reading a stack trace.
2. It finishes the diagnosis first.
3. It saves its position.
   - It writes the cause and the planned fix to the state note.
   - It commits the failing test.
4. It compacts at 55 percent.
5. The next context starts with the test committed and the fix described.

### Measuring the fill level

- `/context` shows what is filling the window, by category, with suggestions.
- Time is a poor proxy for fill level.
  - A session that read three large files can be fuller after ten minutes than another after two hours.
- A hook can read the number on every tool call, which is what [a context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }}) does.

## Compact, rewind or clear

| Command | What it does | Use when |
|---|---|---|
| `/compact <focus>` | Replaces the whole conversation with a summary that keeps what you name | You continue the same task |
| `/rewind`, then "Summarize from here" or "Summarize up to here" | Summarises part of the conversation | One stretch of exploration is no longer needed, the rest is |
| `/clear` | Starts a fresh conversation | You switch to unrelated work; old conversation only costs tokens |

## Steering the summary

`/compact <focus>` steers the summary toward what you name.

```text
/compact Keep: the export-timeout task, the decision to restore the index in
a new migration, and the list of files changed. Drop: the exploration of the
archive table.
```

A focus helps the summary, but it does not replace the state note. The summary can still drop the line you needed.

## The state note

### Fields

| Field | Example |
|---|---|
| Task | Nightly export times out after the migration |
| Done | Cause found: index on `rows.created_at` dropped |
| Decided, with reason | Restore in a new migration, because old migrations already ran in production |
| Next step | Write `test_export_uses_index` |
| Open questions | Does the archive table need the same index? |
| Where things are | Branch `fix-export-index`, files touched |

### Where it lives

- A file in the worktree or the session's notes folder, named for the task.
- Overwrite it at each save, so it always describes the current position.
- History lives in commits and the tracker.

<details markdown="1">
<summary>Why not keep the state in the tracker alone?</summary>

The tracker holds what the tasks are and their status. The state note holds what the session was in the middle of: the half-formed diagnosis, the next command. Both survive compaction; they answer different questions. A short tracker comment can point at the note.

</details>

## Resuming

After compaction the session needs to rebuild its working picture. Two layers make that a fixed routine:

- A resume message, which tells the session what to read and what comes next.
- A `SessionStart` hook on `compact`, which puts the state note into the new context without anyone remembering to.

### A resume message

Written before compacting and pasted or injected after:

```text
Resume: export-timeout fix.
Read: notes/state-export-timeout.md, then the tracker's in-progress list.
Done so far: cause found, failing test committed (3c9d1e0).
Next: write the migration, run the suite.
Rules still in force: no edits to old migrations; ask before touching archive.
```

### A `SessionStart` hook on `compact`

A `SessionStart` hook with the `compact` matcher runs after every compaction, and Claude Code adds what it prints to the new context. The resume then happens whether or not anyone remembers it.

```json
{
  "hooks": {
    "SessionStart": [
      { "matcher": "compact",
        "hooks": [ { "type": "command",
                     "command": "cat notes/state.md; git log --oneline -3" } ] }
    ]
  }
}
```

### Checking the resume worked

1. Restate the task and the next step from the state note.
2. Check the branch and the last commit match the note.
   - `git -C <worktree> log --oneline -1`
3. If they differ, the note is stale: fix it before working.

## Edge cases

| Case | What happens | What to do |
|---|---|---|
| A rule with `paths:` that must hold after compaction | It is gone until a matching file is touched | Drop `paths:` or move it to the root `CLAUDE.md` |
| A long skill in use | Its body is truncated to the per-skill cap | Put the most important instructions near the top of `SKILL.md` |
| A test run in the background | It keeps running across the compaction | Nothing; Claude is reminded it exists |
| A conversation left idle | Idle compaction can compact it | Save state before stepping away on a long task |
| A context overflow compaction cannot clear | An active `/goal` is cleared | Shrink the conversation with `/rewind` or start fresh from the state note |

## Examples from my own harness

- A save-state skill that writes the current state before compaction, and a resume message the next context reads.
  - It records the task, decisions, git state, what is next, and any feedback from me in that session.
  - The resume message lists the plan doc, memory files and in-progress tasks to read.
- A hook-driven supervisor that runs the save before compacting, so saving does not depend on the agent remembering ([a context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }})).

To start: one notes file the agent updates before each compaction, and a `SessionStart` hook on `compact` that prints it.

## Pitfalls and lessons

| Pitfall | What helps |
|---|---|
| Compacting mid-diagnosis | Finish the step, save, then compact |
| Trusting the summary for exact errors and decisions | The state note |
| A resume that depends on the agent remembering | A `SessionStart` hook on `compact` |
| A stale state note | Check the branch and last commit against it before working |
| A `paths:` rule that vanishes | Move must-hold rules to the root `CLAUDE.md` |

## Sources

- What survives compaction, `/compact`, `/autocompact`, `/rewind`, `/clear`, idle compaction: [Explore the context window (Claude Code docs)](https://code.claude.com/docs/en/context-window.md), verified 2026-11-09.
- `SessionStart` hook with the `compact` matcher: [Automate actions with hooks (Claude Code docs)](https://code.claude.com/docs/en/hooks-guide.md), verified 2026-11-09.
- `/goal` cleared on an unrecoverable context overflow: [Keep Claude working toward a goal (Claude Code docs)](https://code.claude.com/docs/en/goal.md), verified 2026-11-09.

{% include page_nav.html %}
