---
title: "Context files after compaction"
parent: "How context files load"
grand_parent: "Context files"
nav_order: 1
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# Context files after compaction

Compaction summarises the conversation to free the window. For context files it is a reset with exceptions:

- some files are read again from disk;
- others are gone until something brings them back;
- so a rule placed in the wrong kind of file can be followed for hours and then silently stop applying.

This page explains what happens and how to place a rule so it holds.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What happens at compaction

![The compaction part of a session: the window fills, the history is summarised, the root CLAUDE.md and unscoped rules are re-read from disk, the nested file and path rule are gone until api is touched again]({{ '/assets/images/04-context-files/loading-sequence.svg' | relative_url }})

*The lower third of this sequence is compaction. Diagram made with archify.*

### Survives

| File | After compaction |
|---|---|
| Project-root `CLAUDE.md` | Re-read from disk |
| Unscoped rules (no `paths:`) | Re-read from disk |
| An instruction you gave only in the conversation | Only as far as the summary keeps it; put a lasting one in `CLAUDE.md` |

### Returns only on a touch

| File | After compaction |
|---|---|
| Subdirectory `CLAUDE.md` | Summarised away; loads again when the agent reads, writes or edits a file in that folder |
| Path rule | Summarised away; loads again when the agent reads, writes or edits a matching file |

### In short

| Kind of file | Loaded at launch | After compaction | Comes back when |
|---|---|---|---|
| Project-root `CLAUDE.md` | Yes | Re-read from disk | Already back |
| Unscoped rule | Yes | Re-read from disk | Already back |
| Subdirectory `CLAUDE.md` | No, on first touch | Gone | The agent reads, writes or edits a file in that folder (`cat` of one file counts) |
| Path rule | No, on first match | Gone | The agent reads, writes or edits a matching file |

The on-demand files come back the same way they came in the first time.

## Failure cases

### The shell write

1. Compaction runs while the agent is working in `api/`.
2. The agent's next action is a shell command that creates a file there.
3. A shell write is not a read, write or edit through the file tools, so `api/CLAUDE.md` does not load.
4. The new file is written without the folder's rules.

```text
  ... compaction ...
● Bash(cat > api/refund_handler.py <<'EOF' ... EOF)
  ⎿  (file written; api/CLAUDE.md not loaded)
```

### The long detour

1. The agent spends the second half of a session in another part of the code.
2. Compaction runs; the first folder's file is summarised away.
3. The agent returns to the first folder by reading or editing a file there.
4. The folder file loads again, so the rules are back before the agent changes anything.

### The summary that keeps a paraphrase

The summary may keep a version of a rule.

| | Text | What is lost |
|---|---|---|
| The rule in `api/CLAUDE.md` | "Handlers return `Result[T, ApiError]`; never raise from a handler." | |
| The summary's paraphrase | "handlers should use typed errors" | The exact type, the prohibition, the reason |

- Do not rely on the paraphrase.
- A check that rejects a handler which raises holds whether or not the rule is in the window.

## Placing a rule so it holds

### Decision steps

1. Must the rule hold every time? Enforce it in the system: a test, a linter or a hook that checks the result. Then write a short line that explains the check, and place that line with the steps below.
2. Does the line apply everywhere in the repository? Put it in the root file, which survives compaction.
3. Does it apply to one kind of file (tests, migrations, docs)? Put it in a path rule with a glob for those files.
4. Does it apply to one folder only? A subdirectory `CLAUDE.md` is enough for guidance that may lapse after a compaction.

### The typed-error rule, three ways

| Placement | Holds after compaction when the agent… |
|---|---|
| A test or linter that rejects a handler without a typed error, run before every commit | …does anything at all; the check runs whatever is in the window |
| Path rule on `api/**` | …reads, writes or edits any file in `api/`, including a new file created with the Write tool; a shell write still skips it |
| `api/CLAUDE.md` | …reads or edits an existing file in `api/` first |

<!-- IMAGE NEEDED: a Claude Code session just after /compact, showing the compaction summary line and the context usage dropping; search: "Claude Code /compact", "Claude Code compaction summary", "context window compaction" -->

### Steering the summary

`/compact <focus>` steers what the summary keeps. Before a planned compaction in the middle of folder work, name the rule:

```text
> /compact keep that every api handler must return a typed error
```

A focus lasts for one compaction in one session. The placements in the table above last across sessions.

## Testing it yourself

### A five-minute experiment

1. Add a harmless, visible rule to `api/CLAUDE.md`: "When you edit a file in api/, end your reply with the word PINEAPPLE."
2. Start a session at the repository root and ask the agent to edit a file in `api/`. The word appears.
3. Run `/compact`.
4. Ask the agent to create a new file in `api/` with a shell command. The word does not appear.
5. Ask it to edit an existing file in `api/` with its edit tool. The word is back.
6. Remove the rule.

| Step | What loaded | What you learn |
|---|---|---|
| 2 | `api/CLAUDE.md` on first touch | On-demand loading works |
| 4 | Nothing new | Compaction removed it and the shell does not bring it back |
| 5 | `api/CLAUDE.md` again | A read or edit restores it |

### Doing the same for a path rule

```markdown
---
paths: ["api/**"]
---
- When you edit a file in api/, end your reply with the word PINEAPPLE.
```

- Save it as `.claude/rules/api.md` and repeat the steps above.
- Step 4 behaves the same: a shell write does not count as a file-tool touch, so the rule stays unloaded.

## In a long autonomous run

A run that compacts several times is the case where placement matters most, because nobody is watching for a dropped rule.

| Rule | Where it should live for a long run |
|---|---|
| Safety rules (what never to run) | Root file and a hook |
| How to test and commit | Root file |
| Conventions for one kind of file | Path rule |
| Folder notes useful while reading code | Subdirectory file |

See [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}) for compaction itself and [Hooks]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) for enforcement.

## Sources

- [Claude Code: the context window](https://code.claude.com/docs/en/context-window.md), verified 2026-11-09: what is re-injected after compaction.
- [Claude Code: how Claude remembers your project](https://code.claude.com/docs/en/memory.md), read 2026-11-09: when nested files and path rules load (including reads with `cat` or `head`), and instructions lost after `/compact`.
- Failure cases and examples are written for this page.

{% include page_nav.html %}
