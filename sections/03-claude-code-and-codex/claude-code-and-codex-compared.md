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
| Hooks | 33 events; command, HTTP, MCP tool, prompt and agent handlers | 14 events; command and MCP tool handlers |
| Sub-agents | Markdown with frontmatter in `.claude/agents/` | TOML in `~/.codex/agents/` or `.codex/agents/`, inheriting the sandbox |
| Goal command | `/goal` | `/goal` |
| Repeating prompt | `/loop` | none |
| Compaction | `/compact`, `/autocompact` | `/compact`, `model_auto_compact_token_limit` |
| Memory | Auto memory, `/memory` | `/memories` |
| Starter context file | `/init` | `/init` |
| Plan mode | `/plan`, or `Shift+Tab` | `/plan` |
| Review the working tree | `/code-review` | `/review` |
| Work in a git worktree | `/batch` units and background sessions run in their own worktrees | `/worktree` |
| Import another setup | none | `/import` reads Claude Code or Cursor setups |

<!-- IMAGE NEEDED: Claude Code and the Codex CLI side by side in two terminals on the same repository; search: "Claude Code vs Codex CLI", "Codex CLI terminal", "Claude Code terminal" -->

## Building block by building block

### Context files

#### Names and walk order

| Step | Claude Code | Codex |
|---|---|---|
| 1 | Managed policy file, then user file `~/.claude/CLAUDE.md` | Global file in `~/.codex` (or `CODEX_HOME`): `AGENTS.override.md` if present, else `AGENTS.md` |
| 2 | Every `CLAUDE.md` from the root down to the working directory, at launch | One file per directory from the project root down to the working directory |
| 3 | `CLAUDE.local.md` and unscoped rules, at launch | In each directory: `AGENTS.override.md`, then `AGENTS.md`, then fallback names |
| 4 | A subdirectory's file when the agent first reads or edits a file there | Stops at 32 KiB in total |

In both, files closer to the working directory come later. Codex documents that they take precedence; Claude Code concatenates the files, and contradicting lines may be followed either way. The whole walk is on [How context files load]({{ '/sections/04-context-files/how-context-files-load.html' | relative_url }}).

#### Size limits

| | Claude Code | Codex |
|---|---|---|
| Documented limit | Suggestion: keep each `CLAUDE.md` under 200 lines | Hard cap: 32 KiB in total across the walk |
| Setting | none | `project_doc_max_bytes` |
| What happens past it | The file still loads; it is followed less reliably | Reading stops at the cap; later files are not read |

> Codex concatenates the files from the root down and stops at the cap. The files nearest the working directory, the most specific ones, are read last, so they are the first to be cut when the total runs over.

#### Reading the other client's file

- Claude Code reads `AGENTS.md` only when no `CLAUDE.md`, `.claude/CLAUDE.md` or `CLAUDE.local.md` exists in the working directory or above it; the Project instructions setting can make it read both.
- Codex adds alternate file names through `project_doc_fallback_filenames`.

### Skills

Both clients use a folder per skill with a `SKILL.md` that has a name and a description in its frontmatter.

1. The model sees every skill's description.
2. When a task matches one, it loads that skill's body.
3. A person can also start a skill by name.

The difference is in who may start one:

| | Claude Code | Codex |
|---|---|---|
| Location | `.claude/skills/<name>/` | `.agents/skills/<name>/` |
| Person-only skill | `disable-model-invocation: true` | `allow_implicit_invocation` set off |

[Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}) covers how to write one.

### Hooks

A hook is a command the client runs on an event, such as before a tool call. Hooks are where a harness enforces what a context file only asks for.

| | Claude Code | Codex |
|---|---|---|
| Events documented | 33 | 14 |
| Handler types | Command, HTTP, MCP tool, prompt, agent | Command and MCP tool; prompt and agent are parsed but skipped |
| Configured in | `hooks` in `settings.json` at each level | `hooks.json` or `[hooks]` in `config.toml`, in `~/.codex/` or `<repo>/.codex/` |
| Blocking a call | Exit code 2, reason on stderr | Exit code 2, reason on stderr, or a JSON `permissionDecision: "deny"` |
| Trust | Project hooks are listed in the workspace trust prompt and apply once you accept it | Non-managed hooks must be trusted; review them with `/hooks` |

The events most harnesses use exist in both:

| Event (both clients) | Typical use |
|---|---|
| `PreToolUse` | Refuse a dangerous command |
| `PostToolUse` | Format or lint the edited file |
| `SessionStart` | Inject current state |
| `UserPromptSubmit` | Add context or refuse a prompt |
| `Stop` | Check the work is finished |

A Codex hook in `<repo>/.codex/hooks.json` that runs a policy script before every shell call:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "python3 \"$(git rev-parse --show-toplevel)/.codex/hooks/policy.py\"", "timeout": 30 }]
      }
    ]
  }
}
```

The shape matches Claude Code's `hooks` block, so one policy script can serve both clients. See [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

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
description = "Reviews a diff for correctness bugs. Use after a change is ready."
model_reasoning_effort = "high"
developer_instructions = """
You review code changes. Read the diff, find bugs that would change behaviour,
and report each with the file, the line and a failing input.
"""
```

| | Claude Code | Codex |
|---|---|---|
| Location | `.claude/agents/` (project), `~/.claude/agents/` (user) | `.codex/agents/` (project), `~/.codex/agents/` (personal) |
| Required | `name`, `description` in the frontmatter, the body as instructions | `name`, `description`, `developer_instructions` |
| Model | `model:` | `model`, `model_reasoning_effort` and related keys |
| Sandbox and approvals | Per profile through `tools:` | Inherited from the parent turn; a profile cannot replace them |

> In Codex, telling a reviewer "do not edit files" in `developer_instructions` is an instruction only; it does not create a read-only sandbox. To enforce read-only, run the parent turn with a read-only sandbox. The same layering applies in Claude Code: a `tools:` list without Edit and Write enforces it, a sentence in the body does not.

The Claude Code file is an illustration of the shape; check each client's reference for the keys your version accepts. Delegation itself is on [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}).

### Configuration format

| | Claude Code | Codex |
|---|---|---|
| Format | JSON | TOML |
| Where | `~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`, managed settings | `config.toml` in `~/.codex` (or `CODEX_HOME`) and in `<repo>/.codex/` |
| Inspect what loaded | `/status`, `claude doctor` | `/status`, `/debug-config` |
| Sub-agent profiles | Markdown in `.claude/agents/` | TOML in `.codex/agents/` |
| Precedence | Managed, command line, local, project, user | See the Codex docs |

The Claude Code levels are on [Configuration and permissions]({{ '/sections/03-claude-code-and-codex/configuration-and-permissions.html' | relative_url }}).

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

1. Someone with a working Claude Code setup runs `/import` in Codex.
2. Codex reads the `CLAUDE.md` and the skills and brings them in.
3. Enforcement needs a second look.
   - A hook written for a Claude Code event has to be matched to a Codex event, and Codex documents fewer.
   - Permission rules in Claude Code's JSON form do not apply to Codex's sandbox settings.
4. Run a known-bad action in Codex, such as the command a hook should refuse, to confirm the check still holds.

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

| Pitfall | What to do |
|---|---|
| Command names and meanings change between releases | Check the reference for your installed version before you script against a command |
| The Codex size cap drops the most specific files first | Keep the shared root file small |
| Hooks and permissions do not move between clients by themselves | Port them on purpose, and test a known-bad action in each client |
| A Codex agent told "do not edit" still has write access | Enforce read-only through the sandbox of the turn that spawns it |
| A hook in an untrusted folder does not run | Trust the folder (Claude Code) or review it with `/hooks` (Codex) |

## Sources

- Claude Code: <https://code.claude.com/docs/en/commands.md>, <https://code.claude.com/docs/en/memory.md>, <https://code.claude.com/docs/en/settings.md>, <https://code.claude.com/docs/en/cross-session-messaging.md>, <https://code.claude.com/docs/en/remote-control.md>, <https://code.claude.com/docs/en/advisor.md>, <https://code.claude.com/docs/en/chrome.md>, verified 2026-11-08 and 2026-11-09.
- Claude Code permissions and workspace trust: <https://code.claude.com/docs/en/permissions.md>, read 2026-11-09.
- Codex: <https://learn.chatgpt.com/docs/agent-configuration/agents-md>, <https://learn.chatgpt.com/docs/developer-commands.md?surface=cli>, <https://learn.chatgpt.com/docs/hooks.md> (locations, events, handler types, blocking), <https://learn.chatgpt.com/docs/agent-configuration/subagents.md> (keys, inheritance), all read 2026-11-09; <https://learn.chatgpt.com/docs/build-skills.md>, verified 2026-11-08.
- The profile files, layout and sequence are illustrations written for this page; the two-family review is from the author's own harness, described conceptually.

{% include page_nav.html %}
