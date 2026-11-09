---
title: "Compaction and saving state"
parent: "Context window and memory"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/11-context-window/detail.html
---

{% include page_nav.html %}

# Compaction and saving state

As of 2026-11-09.

Compaction replaces a session's history with a summary so the session can keep going. It is necessary on any long task and it loses detail every time. This page covers when to compact, what survives, how to steer the summary, how to write state so nothing important depends on the summary, and how to resume. The [section overview]({{ '/sections/11-context-window/' | relative_url }}) is the short version. Sub-topic pages cover [a context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }}) that automates the routine, and [context in Claude Code and Codex]({{ '/sections/11-context-window/claude-code-and-codex-context.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What compaction does

![A sequence diagram: the context hook reports 58% used; the session writes a state note and updates the tracker; it runs /compact with a focus and gets a summary back; it reads its state note back and resumes.]({{ '/assets/images/11-context-window/save-state-and-compact.svg' | relative_url }}){: style="width:100%"}

*Figure: the order that keeps a compaction safe. Made with archify.*

Claude Code compacts automatically near the limit. A model reads the conversation and writes a summary, and the session continues from the summary instead of the full history.

### What survives

| Survives | How |
|---|---|
| Root `CLAUDE.md` | Loaded again after compaction |
| Rules without paths | Loaded again after compaction |
| Auto memory | Loaded again after compaction |
| Files on disk | Untouched; the session can read them again |
| The gist of the conversation | In the summary |

### What is lost

| Lost or blurred | Why it matters |
|---|---|
| Exact error messages and line numbers | The next step often depends on them |
| Decisions and the reason for each | The session may reopen a settled question |
| What was already tried | It may try the same failed fix again |
| Rules you gave in conversation | Unless they were written to a context file |
| Path-scoped rules loaded mid-session | They load again only when matching files are touched |

> The summary can drop any detail. Write anything the next step depends on to a file before compacting.
{: .warning }

## When to compact

### By fill level

As the context window fills, an agent follows instructions less reliably. My own practice is to compact at about 50 to 60 percent of the context window. Other practitioners report a sweet spot below about 300k tokens (unverified practitioner reports). `/autocompact` sets the threshold for automatic compaction in Claude Code.

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

### Measure, do not guess

`/context` shows what is filling the window. A guess based on how long the session has run is not a measurement; a session that read three large files can be fuller after ten minutes than another after two hours.

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

A file in the worktree or the session's notes folder, named for the task. Overwrite it at each save: it describes now, not history. History lives in commits and the tracker.

<details markdown="1">
<summary>Why not keep the state in the tracker alone?</summary>

The tracker holds what the tasks are and their status. The state note holds what the session was in the middle of: the half-formed diagnosis, the next command. Both survive compaction; they answer different questions. A short tracker comment can point at the note.

</details>

## Resuming

After compaction the session needs to rebuild its working picture. A resume message, written before compacting and pasted or injected after, makes that a fixed routine.

```text
Resume: export-timeout fix.
Read: notes/state-export-timeout.md, then the tracker's in-progress list.
Done so far: cause found, failing test committed (3c9d1e0).
Next: write the migration, run the suite.
Rules still in force: no edits to old migrations; ask before touching archive.
```

### Checking the resume worked

The first thing after resuming is to restate the task and next step from the files, and check the branch and last commit match the note. A mismatch means the note is stale; fix it before working.

## Examples from my own harness

A save-state skill writes the current state before compaction, and a resume message format the next context reads. It records the task, decisions, git state, what is next, and any feedback from me in that session. The resume message lists the plan doc, memory files and in-progress tasks to read.

To start: one notes file the agent updates before each compaction, and a resume line that tells it to read that file first.

## Sources

- Compaction, `/compact`, what reloads: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.

{% include page_nav.html %}
