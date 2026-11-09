---
title: "Context files after compaction"
parent: "How context files load"
grand_parent: "Context files"
nav_order: 1
---

{% include page_nav.html %}

# Context files after compaction

As of 2026-11-09. Sub-topic of [How context files load]({{ '/sections/04-context-files/how-context-files-load.html' | relative_url }}) in [Context files]({{ '/sections/04-context-files/' | relative_url }}). Slide: [stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

Compaction summarises the conversation to free the window. For context files it is a reset with exceptions: some files are read again from disk, others are gone until something brings them back. A rule placed in the wrong kind of file can be followed for hours and then silently stop applying. This page explains what happens and how to place a rule so it holds.

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

### Returns only on a touch

| File | After compaction |
|---|---|
| Subdirectory `CLAUDE.md` | Summarised away; loads again when a file tool touches that folder |
| Path rule | Summarised away; loads again when a file tool touches a matching path |

### Why it works this way

Re-reading every file that loaded during the session would put back most of what compaction removed. Re-reading only the always-loaded files keeps the reset cheap, and the on-demand files come back the same way they came in the first time.

## Failure cases

### The shell write

The agent's next action after compaction is a shell command that creates a file. No file tool touched the folder, so the folder's rules never return. This is the most common case.

```text
  ... compaction ...
● Bash(cat > api/refund_handler.py <<'EOF' ... EOF)
  ⎿  (file written; api/CLAUDE.md not loaded)
```

### The long detour

The agent spends the second half of a session in another part of the code. Compaction runs. It returns to the first folder through a file tool, so the folder file loads again. This case is safe.

### The summary that keeps a paraphrase

The summary may keep a version of a rule ("handlers should use typed errors"). A paraphrase is weaker than the rule and has lost its reason. Do not rely on it.

## Placing a rule so it holds

### Decision steps

1. Does the rule apply everywhere in the repository? Put it in the root file.
2. Does it apply to one kind of file (tests, migrations, docs)? Put it in a path rule with a glob for those files.
3. Does it apply to one folder only, and is the folder always entered through file reads? A subdirectory `CLAUDE.md` is enough.
4. Must it hold even when the agent works through the shell? Add a hook or a check that enforces it.

### The typed-error rule, three ways

| Placement | Holds after compaction when the agent… |
|---|---|
| `api/CLAUDE.md` | …reads or edits an existing file in `api/` first |
| Path rule on `api/**` | …touches any file in `api/` with a file tool, a new one included |
| Hook that checks handler return types on commit | …does anything at all; the check runs regardless |

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
| 5 | `api/CLAUDE.md` again | A file tool touch restores it |

### Doing the same for a path rule

Repeat the steps with a rule in `.claude/rules/api.md` with `paths: ["api/**"]`. Step 4 behaves the same: a shell write is not a file-tool touch.

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

- <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-09: what is re-injected after compaction.
- <https://code.claude.com/docs/en/memory.md>, verified 2026-11-08: when nested files and path rules load.
- Failure cases and examples are written for this page.

{% include page_nav.html %}
