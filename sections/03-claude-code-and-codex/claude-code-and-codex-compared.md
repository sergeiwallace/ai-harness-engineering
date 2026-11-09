---
title: "Claude Code and Codex compared"
parent: "Claude Code and Codex basics"
nav_order: 1
redirect_from:
  - /sections/03-claude-code-and-codex/detail.html
---

{% include page_nav.html %}

# Claude Code and Codex compared

As of 2026-11-09. Part of [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}). Slide: [stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

This page sets the two clients side by side, building block by building block: what each one calls it, where it lives on disk, what the defaults and limits are, and what that means when one repository is used with both. It ends with which features need which sign-in.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The feature grid

| Feature | Claude Code | Codex |
|---|---|---|
| Context file | `CLAUDE.md` at user, project and subdirectory levels | `AGENTS.md`, one per directory from the root to the working directory |
| Override file | `CLAUDE.local.md`, personal, not committed | `AGENTS.override.md`, read before `AGENTS.md` in each directory |
| Path-scoped rules | `.claude/rules/` with `paths:` frontmatter | not documented |
| Skills | `.claude/skills/<name>/SKILL.md`; person-only with `disable-model-invocation` | `.agents/skills`; `allow_implicit_invocation` controls model use |
| Hooks | 33 events; command, HTTP, MCP tool, prompt and agent handlers | 14 events |
| Sub-agents | Markdown with frontmatter in `.claude/agents/` | TOML in `.codex/agents/`, inheriting the sandbox |
| Goal command | `/goal` | `/goal` |
| Repeating prompt | `/loop` | none |
| Compaction | `/compact`, `/autocompact` | `/compact`, `model_auto_compact_token_limit` |
| Memory | Auto memory, `/memory` | `/memories` |
| Import another setup | none | `/import` reads Claude Code or Cursor setups |

## Building block by building block

### Context files

#### Names and walk order

Claude Code loads `CLAUDE.md` files from the user level, the project root and every parent directory at launch, and a subdirectory's file when the agent reads or edits a file there. Codex reads a global file from its home directory (`~/.codex` unless `CODEX_HOME` is set), then walks from the project root down to the working directory and includes at most one file per directory, checking `AGENTS.override.md` first. Files closer to the working directory come later and take precedence. The whole walk is on [How context files load]({{ '/sections/04-context-files/how-context-files-load.html' | relative_url }}).

#### Size limits

| | Claude Code | Codex |
|---|---|---|
| Documented limit | Suggestion: keep each `CLAUDE.md` under 200 lines | Hard cap: 32 KiB in total across the walk |
| Setting | none | `project_doc_max_bytes` |
| What happens past it | The file still loads; it is followed less reliably | Reading stops at the cap; later files are not read |

> The Codex cap cuts from the bottom of the walk. The files nearest the working directory, the most specific ones, are the first to be dropped when the total runs over.

#### Reading the other client's file

Claude Code reads `AGENTS.md` only when no `CLAUDE.md` exists in the working directory or above it, unless a setting changes that. Codex adds alternate file names through `project_doc_fallback_filenames`.

### Skills

Both clients use a folder per skill with a `SKILL.md` that has a name and a description in its frontmatter. The model sees the descriptions and loads a skill's body when a task matches. The difference is in who may start one.

| | Claude Code | Codex |
|---|---|---|
| Location | `.claude/skills/<name>/` | `.agents/skills/<name>/` |
| Person-only skill | `disable-model-invocation: true` | `allow_implicit_invocation` set off |

[Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}) covers how to write one.

### Hooks

A hook is a command the client runs on an event, such as before a tool call. Claude Code documents 33 events and five handler types (command, HTTP, MCP tool, prompt and agent). Codex documents 14 events. The events both have, before and after a tool call, at session start, on a prompt and at stop, are the ones most harnesses use. See [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

### Sub-agents

#### Claude Code profile

```markdown
---
name: reviewer
description: Reviews a diff for correctness bugs. Use after a change is ready.
tools: Read, Grep, Glob, Bash
model: opus
---
You review code changes. Read the diff, find bugs that would change behaviour,
and report each with the file, the line and a failing input.
```

#### Codex profile

```toml
# .codex/agents/reviewer.toml
name = "reviewer"
description = "Reviews a diff for correctness bugs."
instructions = """
You review code changes. Read the diff, find bugs that would change behaviour,
and report each with the file, the line and a failing input.
"""
```

The Codex sub-agent inherits the parent's sandbox. The two files are illustrations of the shape; check each client's reference for the exact keys your version accepts. Delegation itself is on [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}).

### Configuration format

Claude Code uses JSON settings files at several levels (see [Configuration and permissions]({{ '/sections/03-claude-code-and-codex/configuration-and-permissions.html' | relative_url }})). Codex uses one TOML file in its home directory, plus profiles.

## One repository, both clients

### The layout

```text
repo/
├── AGENTS.md              shared rules, read by Codex directly
├── CLAUDE.md              "@AGENTS.md" plus Claude-only lines
├── .claude/
│   ├── settings.json      Claude Code permissions and hooks
│   ├── rules/             Claude Code path rules
│   └── agents/            Claude Code sub-agents (Markdown)
├── .codex/
│   └── agents/            Codex sub-agents (TOML)
└── .agents/
    └── skills/            Codex skills
```

### Steps

1. Write the shared rules once, in `AGENTS.md`.
2. Make `CLAUDE.md` an import of it, plus the few lines only Claude Code needs.
3. Keep each client's enforcement in its own place: hooks and permissions do not translate automatically.
4. When the shared file grows, watch the Codex 32 KiB cap as well as the Claude Code line suggestion.

### Moving a setup with /import

Someone with a working Claude Code setup runs `/import` in Codex. It reads the `CLAUDE.md` and the skills and brings them in. What does not carry over by itself is enforcement: a hook written for one client's event names has to be checked against the other's events.

## Using two families on purpose

The two clients run different model families. In my own harness I use that for review, as a choice that suits my setup:

1. Claude Code drives the work and writes the change and its tests.
2. A small wrapper calls Codex with a reviewer configuration.
   - The reviewer gets the diff and the spec only; it never sees the writer's conversation.
   - It returns findings with a file, a line and a failing input.
3. The driver fixes or answers each finding before the change lands.

Same-family review in a fresh context is also a valid choice. [Code review]({{ '/sections/13-code-review/' | relative_url }}) compares the options.

![Sequence: you ask the Claude Code driver for a fix; it writes a failing test and the fix; it sends the diff and the spec to a Codex reviewer with no shared conversation; the reviewer reads the code and returns two findings; the driver fixes one with a new test and justifies the other, then reports back]({{ '/assets/images/03-claude-code-and-codex/two-family-review-sequence.svg' | relative_url }})

*A change written by one model family and reviewed by the other. The reviewer gets the diff and the spec, never the writer's conversation, so it does not inherit the writer's reasoning. Diagram made with archify; the task is an illustration.*

## Availability on cloud providers

For Amazon Bedrock and similar cloud providers:

| Command or feature | On cloud providers |
|---|---|
| Dynamic workflows, `/workflows` | Available |
| `/goal` | Available; the evaluator runs on the session's provider |
| Auto mode | Available with supported models |
| Same-machine session messaging, `/list-agents` | Available from v2.1.248 |
| Cross-machine and cloud session messaging | Not available; needs Remote Control |
| `/remote-control` (`/rc`) | Not available, and API keys are not supported either |
| `/advisor` | Not available; Anthropic API only |
| `/code-review ultra` | Not available; falls back to a local review |
| `/insights` | Available |
| `/deep-research` | Needs the WebSearch tool; whether WebSearch exists on cloud providers is unverified |
| Claude in Chrome, computer use | Not available; need a direct Anthropic plan |

Whether `/loop` needs a sign-in is unverified.

<details markdown="1">
<summary>Why a feature can be missing without an error</summary>

A feature that needs a claude.ai sign-in does not always fail loudly on a cloud provider; it can be absent from the command list. A procedure written on a personal account can then break for a colleague on a work account. Say in each procedure which route it assumes, and test it on that route.

</details>

## Pitfalls and lessons

- Command names and meanings change between releases. Check the reference for your installed version before you script against one.
- The Codex size cap drops the most specific files first. Keep the shared root file small.
- Hooks and permissions do not move between clients by themselves. Port them on purpose.

## Sources

- Claude Code: <https://code.claude.com/docs/en/commands.md>, <https://code.claude.com/docs/en/memory.md>, <https://code.claude.com/docs/en/settings.md>, <https://code.claude.com/docs/en/cross-session-messaging.md>, <https://code.claude.com/docs/en/remote-control.md>, <https://code.claude.com/docs/en/advisor.md>, <https://code.claude.com/docs/en/chrome.md>, verified 2026-11-08 and 2026-11-09.
- Codex: <https://learn.chatgpt.com/docs/agent-configuration/agents-md> (read 2026-11-09), <https://learn.chatgpt.com/docs/developer-commands.md?surface=cli>, <https://learn.chatgpt.com/docs/hooks.md>, <https://learn.chatgpt.com/docs/agent-configuration/subagents.md>, <https://learn.chatgpt.com/docs/build-skills.md>, verified 2026-11-08.
- The profile files, layout and sequence are illustrations written for this page; the two-family review is from the author's own harness, described conceptually.

{% include page_nav.html %}
