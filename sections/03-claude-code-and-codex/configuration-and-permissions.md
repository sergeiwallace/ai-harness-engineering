---
title: "Configuration and permissions"
parent: "Claude Code and Codex basics"
nav_order: 3
---

{% include page_nav.html %}

# Configuration and permissions

As of 2026-11-09. Part of [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}). Slide: [stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

Settings decide what an agent may do without asking, which model it starts on, which hooks run and which environment it gets. This page covers where Claude Code's settings files live, which one wins when two disagree, how permission rules are written and combined, and how to check what actually loaded.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Settings files

### The four files

| Scope | File | Who it affects | Use it for |
|---|---|---|---|
| User | `~/.claude/settings.json` | You, every project on this machine | Your preferences and your own permission rules |
| Shared project | `.claude/settings.json` | Everyone in the repository, once committed | Team permissions, hooks, plugins, environment |
| Project local | `.claude/settings.local.json` | You, this project only; kept out of git | Personal overrides, testing before you share |
| Managed | `managed-settings.json` and other managed sources | Everyone your organization deploys it to | Security policy and compliance |

Installing Claude Code creates none of them. Claude Code writes the user file the first time you change an option in `/config`, and the local file the first time you answer a permission prompt with "Yes, and don't ask again".

### Where the files sit

```text
~/.claude/
└── settings.json              user: you, every project
repo/
└── .claude/
    ├── settings.json          shared project: committed
    └── settings.local.json    project local: you, kept out of git
<system directory>/
└── managed-settings.json      managed: your organization
```

### Local file in a repository

In a git repository the local file lives at the repository root even if you start in a subdirectory, and in a worktree it lives at the main checkout's root. Claude Code adds it to your global git excludes the first time it writes it.

## Precedence

### Highest first

1. Managed settings. Nothing in your files or on the command line overrides a managed key.
2. Command line, `--settings <file-or-json>`, for one session.
3. Project local.
4. Shared project.
5. User.

Environment variables are not a level in this stack. Which of a variable and a key wins is decided per pair: `ANTHROPIC_MODEL` exported in your shell applies over the `model` key in any file.

### Lists merge

A list key set in several files, such as `permissions.allow`, is combined, so each file adds entries without removing another's. A few model-list keys follow their own rules.

### A team value and a personal one

The team's `.claude/settings.json` sets `"model": "claude-sonnet-5"`. You want Opus in this project only:

```json
// .claude/settings.local.json (comments are not allowed in the real file)
{ "model": "claude-opus-5-5" }
```

Project local sits above shared project, so only your sessions change. To try it for one session without writing a file, start with `claude --settings '{"model": "claude-opus-5-5"}'`.

> Settings files are strict JSON. A `//` comment or a trailing comma is a syntax error, reported as a Settings Error at the next start. The comment above is for the page only.

## Permissions

### Rule syntax

A rule names a tool and, in parentheses, what it may do.

```json
{
  "permissions": {
    "allow": ["Bash(npm run lint)", "Bash(npm run test *)"],
    "ask":   ["Bash(git push *)"],
    "deny":  ["Read(./.env)", "Read(./.env.*)"]
  }
}
```

| Rule | Meaning |
|---|---|
| `Bash(npm run lint)` | That exact command |
| `Bash(npm run test *)` | Any command starting with `npm run test ` |
| `Read(./.env)` | Reading that file |

### How rules combine

![Sequence of three tool calls: make test matches allow and runs after the PreToolUse hook; git push matches ask, you answer yes and it runs; reading .env matches deny and is refused]({{ '/assets/images/03-claude-code-and-codex/permission-check-sequence.svg' | relative_url }})

*Three tool calls through the merged rules and a PreToolUse hook. Diagram made with archify; the rules are the example file below.*

#### Allow, ask and deny

Rules from every file merge into one set, and a more restrictive rule from any file is not outvoted by an allow rule elsewhere. An `allow` in your local file does not outrank an `ask` from the project or managed file. If you chose "don't ask again" and still get asked, that is the reason.

#### Workspace trust

Committed `permissions.allow` rules, extra directories and most environment values in `.claude/settings.json` apply only after each person trusts the folder. `deny` and `ask` rules apply at once. A repository you clone cannot widen your permissions until you say so.

### Permission modes

| Mode | What changes |
|---|---|
| Default | Asks for anything not allowed |
| Plan | Reads and plans, makes no edits |
| Auto | A classifier approves routine actions in long runs |
| Bypass | Asks for nothing |

`auto` and `bypassPermissions` as a default mode take effect only from user or managed settings, or `--permission-mode` for one session; a project or local file cannot set them. [Human gates]({{ '/sections/14-human-gates/' | relative_url }}) covers where a person should stay in the loop.

### A safe default for a team repository

1. Allow the read-only and test commands the team runs all day.
2. Ask before anything that leaves the machine: pushes, deploys, package publishes.
3. Deny reading secrets files.
4. Commit it as `.claude/settings.json`; each teammate trusts the folder once.

```json
{
  "permissions": {
    "allow": ["Bash(git status)", "Bash(git diff *)", "Bash(make test *)", "Bash(make lint)"],
    "ask":   ["Bash(git push *)", "Bash(make deploy *)", "Bash(npm publish *)"],
    "deny":  ["Read(./.env)", "Read(./.env.*)", "Read(./secrets/**)"]
  }
}
```

> A permission rule is a gate on the client's tools. A command that wraps another (`bash -c "..."`, a script) can do things the rule did not name. For a rule that must hold, add a hook that inspects the command. See [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

## Hooks in settings

Hooks are configured in the same files, under `hooks`, keyed by event, with a matcher for the tool name.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }]
      }
    ]
  }
}
```

The hook receives the tool call as JSON on standard input, so the command reads the edited file's path from it. Claude Code reloads settings files when they change, so an edited hook applies in the running session. `/hooks` shows every hook in effect and which file it came from.

## Checking what loaded

| Question | How to answer it |
|---|---|
| Which files loaded? | `/status`, the `Setting sources` line |
| Which entries were rejected? | `claude doctor` |
| Which hooks run? | `/hooks` |
| Is a managed policy in force? | `/status` names the managed source |

<details markdown="1">
<summary>Why a setting can be ignored</summary>

- A higher level sets the same key.
- A security key keeps its stricter value from any level.
- The file cannot set that value (for example a default mode of `auto` from a project file).
- The file is broken JSON and was skipped. In a `claude -p` run there is no dialog, so check `claude doctor` afterwards.

</details>

## Codex configuration

Codex keeps its settings in a TOML file in its home directory (`~/.codex` unless `CODEX_HOME` is set), alongside its global `AGENTS.md`. Sub-agent profiles are TOML files in `.codex/agents/` and inherit the parent's sandbox. The context-file settings `project_doc_max_bytes` (default 32 KiB) and `project_doc_fallback_filenames` live there too. See [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}).

## Pitfalls and lessons

- A personal "don't ask again" lands in the local file and cannot override a project `ask` rule.
- A setting written to a file you cannot write to applies for the session and is gone in the next.
- A cloud session reads committed project settings only; your user and local files stay on your machine.

## Sources

- Settings files, precedence, merging, trust and troubleshooting: <https://code.claude.com/docs/en/settings.md>, read 2026-11-09.
- Permissions and modes: <https://code.claude.com/docs/en/permissions.md>, <https://code.claude.com/docs/en/permission-modes.md>, verified 2026-11-09.
- Codex: <https://learn.chatgpt.com/docs/agent-configuration/agents-md>, read 2026-11-09; <https://learn.chatgpt.com/docs/agent-configuration/subagents.md>, verified 2026-11-08.
- Example files are illustrations written for this page.

{% include page_nav.html %}
