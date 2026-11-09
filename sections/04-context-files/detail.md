---
title: "Detail"
parent: "Context files"
nav_order: 1
---

# Context files in detail

As of 2026-11-09.

## Purpose

Explain how context files load and how to organise them.

## Key points

- `CLAUDE.md` (Claude Code) and `AGENTS.md` (Codex and other tools) hold instructions the client loads into context.
- They are hierarchical. In Claude Code a user-level file applies everywhere and a project file applies to one repository. Files in the directories above the working directory load at launch. A file in a subdirectory loads when the agent reads or edits a file there.
- A context file can import another with `@path` syntax, up to four levels deep.
- Path-scoped rules live in `.claude/rules/`. A `paths:` list in a rule's frontmatter makes it load only when the agent touches a matching file. After compaction, path-scoped rules load again on demand.
- Codex walks from the project root to the working directory, reads one `AGENTS.md` per directory, checks `AGENTS.override.md` first, and stops at 32 KiB in total by default.
- Claude Code reads `AGENTS.md` only when no `CLAUDE.md` exists in the working directory or above it, unless a setting changes that.
- Claude Code's docs suggest keeping each `CLAUDE.md` under 200 lines.
- Hub and spoke: the context file holds one line per rule with a link to a procedure document that holds the detail. The agent reads the procedure when a task needs it.

## Examples from my own harness

- Three layers of context: one file shared by all my projects, one file per repository, and a section for each machine I work on. The shared file is assembled from smaller source parts by a script, checked against a size budget, and installed on every machine.
- Path-scoped rules for tests, for documentation and for writing skills.
- Procedure documents as the spokes, each named by one line in the shared file.

To start: one hand-written `CLAUDE.md` per repository.

## Pitfalls and lessons

- Context files grow faster than they are pruned, and a long file gets followed less reliably.
- Linking out moves the size problem into the procedures. A procedure that grows to thousands of lines is no easier to follow than a long context file.
- Path-scoped rules load only when a file tool touches a matching path. An agent that works only through shell commands never sees them.
- Context files go stale when the mechanisms they describe change. Sweep them like code.
- Reflexivity example: when an agent repeats a mistake after a correction, the harness is updated in the same session so the next session starts with the fix. A context-file line is the cheapest update. When the mistake can be detected in code, a hook or a check is the stronger one (see [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }})).

## Sources

- Claude Code loading order, imports, `AGENTS.md` behaviour, path rules and the 200-line suggestion: <https://code.claude.com/docs/en/memory.md>, verified 2026-11-08.
- Path rules after compaction: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.
- Codex `AGENTS.md`: <https://learn.chatgpt.com/docs/agent-configuration/agents-md>, verified 2026-11-08.
