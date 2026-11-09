---
title: "How context files load"
parent: "Context files"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/04-context-files/detail.html
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# How context files load

A rule only works if it is in the window when the agent needs it. This page explains, for Claude Code and then Codex:

- which files load, from where, and at what moment;
- how imports, comments and exclusions change what loads;
- what happens when two files disagree, and when a file is too long;
- how to run one repository with both clients, and how to check what loaded.

What changes at compaction has its own sub-topic: [Context files after compaction]({{ '/sections/04-context-files/after-compaction.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Claude Code

### The levels

| Level | Location | Applies to | Loads |
|---|---|---|---|
| Managed policy | Linux and WSL `/etc/claude-code/CLAUDE.md`; macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`; Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | Every user on the machine | At launch; cannot be excluded |
| User | `~/.claude/CLAUDE.md` and `~/.claude/rules/` | Every project on the machine | At launch |
| Project | `CLAUDE.md` at the repository root, or `.claude/CLAUDE.md` | One repository, shared with the team | At launch |
| Local | `CLAUDE.local.md` | One repository, one person; add it to `.gitignore` | At launch |
| Parent directories | `CLAUDE.md` in any directory above the working directory | Work started below them | At launch |
| Unscoped rule | `.claude/rules/*.md` without `paths:` | The repository | At launch, with the priority of `.claude/CLAUDE.md` |
| Subdirectory | `CLAUDE.md` in a folder below the working directory | Work in that folder | When the agent reads, writes or edits a file there |
| Path rule | `.claude/rules/*.md` with `paths:` | Matching files only | When the agent reads, writes or edits a matching file |

An organization can also put managed instructions in the `claudeMd` key of its managed settings.

### Launch and on demand

![Sequence of one session showing launch loading, on-demand loading on first touch, a shell write that loads nothing, and compaction]({{ '/assets/images/04-context-files/loading-sequence.svg' | relative_url }})

*Launch loads everything from the managed and user files down to the working directory. Everything below the working directory waits for the agent to touch a file there. Diagram made with archify.*

#### At launch

1. Claude Code starts in the working directory, say `repo/api`.
2. It loads the managed and user files, then every `CLAUDE.md` and `CLAUDE.local.md` from the filesystem root down to `repo/api`, then the unscoped rules.
3. The files are concatenated, root first; nothing overrides anything.
   - `repo/CLAUDE.md` appears before `repo/api/CLAUDE.md`, so instructions nearer the working directory are read last.
   - Within each directory, `CLAUDE.local.md` comes after `CLAUDE.md`.

#### On demand

| What the agent does | Loads |
|---|---|
| Reads, writes or edits a file in `repo/web/` with its file tools | `repo/web/CLAUDE.md`, once |
| Views a single file there with `cat` or `head` | The same; it counts as a read |
| Reads, writes or edits a file matching a rule's `paths` | That rule |
| Writes a file with a shell command (`cat > web/new.tsx`, a generator, `sed -i`) | Nothing |

When an on-demand file loads, a `Loaded` line with its path appears in the terminal.

### Imports

A context file can import another with `@path`.

```markdown
<!-- repo/CLAUDE.md -->
@docs/conventions.md
@~/.claude/my-preferences.md

- Test: `make test`.
- See `@README` for setup.   (in backticks: not imported)
```

| Rule | Detail |
|---|---|
| Path resolution | Relative paths resolve from the folder of the file that contains the import |
| Depth | Imported files can import others, up to four hops |
| Code | `@path` inside a code span or fenced block is left as text |
| Spaces in a path | Escape each with a backslash; a quoted path is not imported |
| External imports | An import that resolves outside the working directory needs your approval once per project |
| Cost | An imported file loads with its parent; importing organises a file without making it cheaper |

| Use an import for | Avoid it for |
|---|---|
| Rules shared between two clients (`@AGENTS.md`) | Long procedures; an import loads them every session |
| Personal instructions shared across worktrees (`@~/.claude/...`), since a gitignored `CLAUDE.local.md` exists only in one worktree | Files that change often |

### Comments and exclusions

- Block-level HTML comments (`<!-- maintainer notes -->`) are stripped before the content reaches the model; use them for notes to people. Comments inside code blocks are kept.
- In a monorepo, `claudeMdExcludes` skips other teams' files by path or glob, set in any settings layer (often `.claude/settings.local.json`):

```json
{
  "claudeMdExcludes": [
    "**/other-team/CLAUDE.md",
    "**/legacy/.claude/rules/**"
  ]
}
```

Patterns match absolute paths; the managed policy file cannot be excluded.

### Conflicts

All the files are concatenated, so two lines can disagree. When they do, the model may follow either one.

- Write files so they do not need to win: a subdirectory file adds rules for its folder.
- If a folder needs the opposite of a root rule, say so in both places:

```markdown
<!-- repo/CLAUDE.md -->
- Indent with 4 spaces, except in `legacy/`, which keeps tabs.

<!-- repo/legacy/CLAUDE.md -->
- This folder keeps tabs; do not reformat it.
```

- If the difference matters, a formatter configuration per folder enforces it and the lines explain it.
- `/doctor prompt-audit` looks for files that contradict each other.

### Size limits

| File | Limit | Past the limit |
|---|---|---|
| Any `CLAUDE.md` | Target under 200 lines | Loads in full, costs more context and is followed less reliably; Claude Code warns at startup and in `/status` |
| Files loaded at launch | 4 MiB | The file is skipped entirely |
| Subdirectory files, their imports, and rules loaded on demand | 40,000 characters on a 200K context window (more on a 1M window) | Only the start is loaded, with a note telling the agent to read the rest |
| Many files together | A combined limit at session start | A warning, even when each file is within its own length |

Check a file's length with `wc -m path/to/CLAUDE.md`. Each `CLAUDE.md`, rules file and import counts as a separate file.

### A monorepo

```text
repo/
├── CLAUDE.md                 build, test, commit rules for everyone
├── .claude/rules/
│   ├── tests.md              paths: ["**/tests/**"]
│   └── migrations.md         paths: ["services/*/migrations/**"]
├── services/
│   ├── billing/CLAUDE.md     money in integer cents; never floats
│   └── search/CLAUDE.md      reindex after a schema change
└── web/CLAUDE.md             component and styling conventions
```

| The agent is working on (session started at `repo/`) | Loaded |
|---|---|
| `web/Button.tsx` | Root file, then `web/CLAUDE.md` on the first touch |
| `services/billing/tests/test_refund.py` | Root file, `services/billing/CLAUDE.md`, `tests.md` |
| `services/search/migrations/0042.sql` | Root file, `services/search/CLAUDE.md`, `migrations.md` |

## Codex

### The walk

1. Global: in its home directory (`~/.codex` unless `CODEX_HOME` is set), Codex uses `AGENTS.override.md` if present, otherwise `AGENTS.md`; only the first non-empty file at this level.
2. Project: from the project root down to the current directory, in each directory it checks `AGENTS.override.md`, then `AGENTS.md`, then any fallback names, and includes at most one file per directory.
3. Merge: the files are concatenated from the root down, so files closer to the current directory come later and take precedence.
4. Cap: reading stops at 32 KiB in total by default; `project_doc_max_bytes` changes it.

### What the cap does

The files are read root first, so in a deep tree with a large root file, the files nearest the working directory, the most specific ones, are the first to be cut.

| Root file | Folder files | What Codex reads in `a/b/c` |
|---|---|---|
| 4 KiB | 3 x 2 KiB | All four |
| 30 KiB | 3 x 2 KiB | The root file and part of the next; `c`'s rules are lost |

The sizes are illustrative.

### Fallback names

`project_doc_fallback_filenames` in the Codex configuration adds other file names to look for.

- Use it for a repository that already keeps instructions under another name.
- `/init` in Codex writes an `AGENTS.md` scaffold.

## Both clients on one repository

### What Claude Code does with AGENTS.md

| Your repository has | Claude Code reads |
|---|---|
| An `AGENTS.md`, and no `CLAUDE.md`, `.claude/CLAUDE.md` or `CLAUDE.local.md` in the working directory or above | Every `AGENTS.md` from the working directory up, and a subdirectory's `AGENTS.md` when it opens a file there |
| An `AGENTS.md` and any of those three files | The `CLAUDE.md` files only |
| A `CLAUDE.md` that imports `@AGENTS.md` | Both, through the import |

- Your `~/.claude/CLAUDE.md`, the managed file and `.claude/rules/` do not count for that check; they load alongside `AGENTS.md`.
- A `CLAUDE.local.md` does count, so adding one for personal notes stops Claude Code reading `AGENTS.md`.
- The Project instructions setting in `/config` changes the rule: `claude-md-or-agents-md` (the default), `claude-md-and-agents-md` (both), or `managed-only`.
- Reading `AGENTS.md` directly needs Claude Code v2.1.277 or later; in sessions that cannot read it, import it from a `CLAUDE.md`.

### The shared layout

```markdown
<!-- CLAUDE.md -->
@AGENTS.md
- Claude Code only: run /code-review before opening a pull request.
```

| | Codex | Claude Code |
|---|---|---|
| Reads | `AGENTS.md` directly | `CLAUDE.md`, which imports `AGENTS.md` |
| Size limit to watch | 32 KiB total | 200-line target per file |
| Path rules | not documented | `.claude/rules/` |

See [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}).

## Checking what loaded

| Question | Claude Code |
|---|---|
| Which files loaded at launch? | `/context all`, the list under Memory files |
| Did a subdirectory file load? | A `Loaded` line in the terminal; it does not appear under Memory files |
| Which files exist, and was `AGENTS.md` read? | `/memory` |
| When and why did each file load? | An `InstructionsLoaded` hook can log it |
| Is a file too long? | Startup warnings and `/status` |

<!-- IMAGE NEEDED: the Claude Code /memory menu listing the user and project CLAUDE.md files and the auto memory folder; search: "Claude Code /memory command", "CLAUDE.md memory files" -->

<details markdown="1">
<summary>A test you can run in two minutes</summary>

1. Create `api/CLAUDE.md` from your shell (not through the agent) with a harmless rule: "End every reply about this folder with the word PINEAPPLE."
2. Start a session at the repository root and ask a question about `api/` without opening a file. The word does not appear.
3. Ask the agent to read a file in `api/`. A `Loaded` line appears, and the next reply ends with the word.
4. Remove the file.

</details>

## Sources

- [Claude Code: how Claude remembers your project](https://code.claude.com/docs/en/memory.md), read 2026-11-09: levels, load order, on-demand loading, imports, comments, `claudeMdExcludes`, conflicts, size limits, `AGENTS.md` handling, checking and troubleshooting.
- [Codex: custom instructions with AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md), read 2026-11-09: the Codex walk, override, cap and fallback names; Codex `/init`: [Codex: CLI slash commands](https://learn.chatgpt.com/docs/developer-commands.md?surface=cli), read 2026-11-09.
- The monorepo, sizes and test are illustrations written for this page.

{% include page_nav.html %}
