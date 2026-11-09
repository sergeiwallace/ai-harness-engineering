---
title: "How context files load"
parent: "Context files"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/04-context-files/detail.html
---

{% include page_nav.html %}

# How context files load

As of 2026-11-09. Part of [Context files]({{ '/sections/04-context-files/' | relative_url }}). Slide: [stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

A rule only works if it is in the window when the agent needs it. This page explains, for Claude Code and then Codex, which files load, in what order, at what moment, and what happens when two say different things. What changes at compaction has its own sub-topic: [Context files after compaction]({{ '/sections/04-context-files/after-compaction.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Claude Code

### The levels

| Level | Location | Applies to | Loads |
|---|---|---|---|
| User | `~/.claude/CLAUDE.md` | Every project on the machine | At launch |
| Project | `CLAUDE.md` at the repository root, or `.claude/CLAUDE.md` | One repository, shared with the team | At launch |
| Local | `CLAUDE.local.md` | One repository, one person, not committed | At launch |
| Parent directories | `CLAUDE.md` in any directory above the working directory | Work started below them | At launch |
| Subdirectory | `CLAUDE.md` in a folder below the working directory | Work in that folder | When the agent reads or edits a file there |
| Path rule | `.claude/rules/*.md` with `paths:` | Matching files only | When a file tool touches a matching path |
| Unscoped rule | `.claude/rules/*.md` without `paths:` | The repository | At launch, like a context file |

### Launch and on demand

![Sequence of one session showing launch loading, on-demand loading on first touch, a shell write that loads nothing, and compaction]({{ '/assets/images/04-context-files/loading-sequence.svg' | relative_url }})

*Launch loads everything from the user file down to the working directory. Everything below the working directory waits for a file tool to touch it. Diagram made with archify.*

#### At launch

Claude Code starts from the working directory and loads every `CLAUDE.md` from there up to the root, plus the user file, the local file and every unscoped rule. A session started in `repo/api` therefore loads `repo/CLAUDE.md` and `repo/api/CLAUDE.md` at once.

#### On demand

A session started in `repo` does not load `repo/api/CLAUDE.md` until the agent reads or edits a file in `api/`. A path rule loads the first time a file tool (read, edit, write) touches a path that matches one of its globs.

#### What does not trigger a load

A shell command that writes a file (`cat > api/new.py`, a code generator, `sed -i`) is not a file tool. It loads no subdirectory file and no path rule.

### Imports

A context file can import another with `@path` syntax, up to four levels deep.

```markdown
<!-- repo/CLAUDE.md -->
@docs/conventions.md
@~/.claude/my-preferences.md

- Test: `make test`.
```

| Use an import for | Avoid it for |
|---|---|
| Rules shared between two clients (`@AGENTS.md`) | Long procedures; an import loads them on every turn |
| A personal file kept outside the repository | Pulling in files that change often |

> An import is not a link. The imported file loads in full, wherever the import sits. To keep detail out of the window until it is needed, link to it with a trigger instead (see [Hub and spoke and path rules]({{ '/sections/04-context-files/hub-and-spoke-and-path-rules.html' | relative_url }})).

### Conflicts

When two files disagree, the more specific one is read later, nearer to the task.

- Write files so they do not need to win: a subdirectory file adds rules for its folder.
- If a folder needs the opposite of a root rule, say so in both places:

```markdown
<!-- repo/CLAUDE.md -->
- Indent with 4 spaces, except in `legacy/`, which keeps tabs.

<!-- repo/legacy/CLAUDE.md -->
- This folder keeps tabs; do not reformat it.
```

- If the conflict matters, a formatter config per folder enforces it and the lines explain it.

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

| The agent is working on | Loaded |
|---|---|
| `web/Button.tsx` | root file, `web/CLAUDE.md` |
| `services/billing/tests/test_refund.py` | root file, `services/billing/CLAUDE.md`, `tests.md` |
| `services/search/migrations/0042.sql` | root file, `services/search/CLAUDE.md`, `migrations.md` |
## Codex

### The walk

1. Global: in its home directory (`~/.codex` unless `CODEX_HOME` is set), Codex uses `AGENTS.override.md` if present, otherwise `AGENTS.md`; only the first non-empty file at this level.
2. Project: from the project root down to the current directory, in each directory it checks `AGENTS.override.md`, then `AGENTS.md`, then any fallback names, and includes at most one file per directory.
3. Merge: the files are concatenated from the root down, so files closer to the current directory come later and take precedence.
4. Cap: reading stops at 32 KiB in total by default; `project_doc_max_bytes` changes it.

### What the cap does

The cap counts from the top of the walk. In a deep tree with a large root file, the files nearest the working directory, the most specific ones, are the first to be cut.

| Root file | Folder files | What Codex reads in `a/b/c` |
|---|---|---|
| 4 KiB | 3 x 2 KiB | All four |
| 30 KiB | 3 x 2 KiB | The root file and part of the next; `c`'s rules are lost |

The sizes are illustrative.

### Fallback names

`project_doc_fallback_filenames` in the Codex configuration adds other file names to look for.

- Use it for a repository that already keeps instructions under another name.
- In each directory Codex checks `AGENTS.override.md`, then `AGENTS.md`, then the fallback names, and takes at most one file.

## Both clients on one repository

Claude Code reads `AGENTS.md` only when no `CLAUDE.md` exists in the working directory or above it, unless a setting changes that. The simplest shared layout keeps `AGENTS.md` as the source and a short `CLAUDE.md` that imports it:

```markdown
<!-- CLAUDE.md -->
@AGENTS.md
- Claude Code only: run /code-review before opening a pull request.
```

| | Codex | Claude Code |
|---|---|---|
| Reads | `AGENTS.md` directly | `CLAUDE.md`, which imports `AGENTS.md` |
| Size limit to watch | 32 KiB total | 200-line suggestion per file |
| Path rules | not documented | `.claude/rules/` |

See [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}).

## Checking what loaded

- `/memory` in Claude Code lists the context files in effect and opens them for editing.
- `/context` shows how much of the window the memory files take.

<details markdown="1">
<summary>A test you can run in two minutes</summary>

Put a harmless, distinctive rule in a subdirectory file, such as "End every reply about this folder with the word PINEAPPLE". Start a session at the root and ask a question about the folder without opening a file: the word does not appear. Ask the agent to read a file in the folder, then ask again: it does. You have just watched an on-demand load. Remove the rule afterwards.

</details>

## Sources

- <https://code.claude.com/docs/en/memory.md>, verified 2026-11-08: levels, imports, `AGENTS.md` behaviour, path rules.
- <https://learn.chatgpt.com/docs/agent-configuration/agents-md>, read 2026-11-09: the Codex walk, override, cap and fallback names.
- The monorepo, sizes and test are illustrations written for this page.

{% include page_nav.html %}
