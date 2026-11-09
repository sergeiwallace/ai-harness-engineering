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

| You start Claude Code in | It reads and writes `settings.local.json` at |
|---|---|
| The repository root | The repository root |
| A subdirectory of the repository | The repository root, so an approval applies to the whole repository |
| A git worktree | The main checkout's root |
| A folder outside git, or when the root is your home directory | Next to `.claude/settings.json` |

- Claude Code adds `**/.claude/settings.local.json` to your global git excludes the first time it writes the file.
- A file you created by hand needs a `.gitignore` entry until then.

## Precedence

### Highest first

1. Managed settings. Nothing in your files or on the command line overrides a managed key.
2. Command line, `--settings <file-or-json>`, for one session.
3. Project local.
4. Shared project.
5. User.

Environment variables are not a level in this stack. Which of a variable and a key wins is decided per pair: `ANTHROPIC_MODEL` exported in your shell applies over the `model` key in any file.

### Lists merge

A list key set in several files is combined: each file adds entries without removing another's.

```text
~/.claude/settings.json          allow: Bash(git status)
.claude/settings.json            allow: Bash(make test *)
.claude/settings.local.json      allow: Bash(pytest *)
-> in effect                     allow: Bash(git status), Bash(make test *), Bash(pytest *)
```

- Scalar keys such as `model` take the highest level's value.
- A few model-list keys, such as `fallbackModel` and `availableModels`, follow their own rules.

### A team value and a personal one

The team's `.claude/settings.json` sets `"model": "claude-sonnet-5"`. You want Opus in this project only:

```json
// .claude/settings.local.json (comments are not allowed in the real file)
{ "model": "claude-opus-5-5" }
```

Project local sits above shared project, so only your sessions change. To try it for one session without writing a file, start with `claude --settings '{"model": "claude-opus-5-5"}'`.

> Settings files are strict JSON. A `//` comment or a trailing comma is a syntax error, reported as a Settings Error at the next start. The comment above is for the page only.

## Permissions

### What asks by default

Before you write any rule, Claude Code already asks for some actions and not others.

| Tool type | Example | Asks in the default mode | "Yes, and don't ask again" lasts |
|---|---|---|---|
| Read-only | File reads, Grep | No, inside the working and added directories | not needed |
| Bash commands | Shell execution | Yes, except a built-in set of read-only commands | Permanently, per repository and command |
| File modification | Edit, Write | Yes | Until the session ends |
| Web fetch | WebFetch | Yes, except preapproved documentation domains | Permanently, per repository and domain |
| Web search | WebSearch | Yes | Permanently, per repository |

<!-- IMAGE NEEDED: a Claude Code permission prompt for a Bash command, showing the command and the Yes / Yes and don't ask again / No options; search: "Claude Code permission prompt", "Claude Code allow command dialog" -->

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
| `Bash(npm run test *)` | `npm run test` alone, or followed by anything |
| `Bash(npm run test:*)` | The same; `:*` is another way to write a trailing wildcard |
| `Read(./.env)` | Reading that file |

### Wildcards and compound commands

Claude Code matches everything before the first `*` as written, and it splits compound commands before matching.

| Rule | Allows or catches | Does not |
|---|---|---|
| allow `Bash(git log *)` | `git log`, `git log --oneline main` | `git push` |
| allow `Bash(npm test *)` | `npm test -- --watch` | `npm test && curl evil.sh`; every part of a compound command must be allowed |
| ask `Bash(git clean *)` | `cd /tmp && git clean -f`, and `echo "$(git clean -f)"`; ask and deny rules apply to any subcommand | |
| deny `Bash(rm *)` | `rm -rf build/` | `/bin/rm -rf build/`, `bash -c 'rm -rf build/'` |

- The separators Claude Code splits on are `&&`, `||`, `;`, `|`, `|&`, `&` and newlines.
- Approving a compound command with "don't ask again" saves one rule per subcommand.

> A deny rule matches the command text the agent usually writes; the same program called another way (a full path, `bash -c`, a script) is not matched. For a prohibition that must hold, a PreToolUse hook that inspects the command does the enforcing; see [Hooks in settings](#hooks-in-settings).

### How rules combine

![Sequence of three tool calls: make test matches allow and runs after the PreToolUse hook; git push matches ask, you answer yes and it runs; reading .env matches deny and is refused]({{ '/assets/images/03-claude-code-and-codex/permission-check-sequence.svg' | relative_url }})

*Three tool calls through the merged rules and a PreToolUse hook. Diagram made with archify; the rules are the example file below.*

#### Allow, ask and deny

Rules from every file merge into one set, and a more restrictive rule from any file is not outvoted by an allow rule elsewhere. An `allow` in your local file does not outrank an `ask` from the project or managed file. If you chose "don't ask again" and still get asked, that is the reason.

#### Workspace trust

| Committed in `.claude/settings.json` | Applies |
|---|---|
| `permissions.allow`, `permissions.additionalDirectories`, most `env` values | After each person accepts the workspace trust prompt |
| Hooks and helper commands | Listed in the trust prompt; after it is accepted |
| `deny` and `ask` rules | At once |

A repository you clone cannot widen your permissions until you say so.

### Permission modes

| Mode | Label | What changes |
|---|---|---|
| `default` | Manual | Asks on first use of each tool that needs approval |
| `acceptEdits` | | Accepts file edits and common filesystem commands (`mkdir`, `touch`, `mv`, `cp`) inside the working directories |
| `plan` | Plan | Reads and runs read-only commands to explore; does not edit source files |
| `auto` | | No routine prompts; a background classifier checks actions such as shell commands and network requests against your request |
| `dontAsk` | | Denies every call that would otherwise prompt; allowed and approval-free actions still run |
| `bypassPermissions` | | Skips prompts, except for actions no mode auto-approves |

- `Shift+Tab` cycles the modes in a session; `--permission-mode <mode>` starts in one.
- `auto` and `bypassPermissions` as a default take effect only from user or managed settings, or `--permission-mode` for one session; a project or local file cannot set them.
- `dontAsk` suits unattended runs where anything unexpected should fail closed.
- [Human gates]({{ '/sections/14-human-gates/' | relative_url }}) covers where a person should stay in the loop.

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

## Hooks in settings

Hooks are configured in the same settings files, under `hooks`.

| Key | Holds | Example |
|---|---|---|
| Event name | A list of matcher groups for that event | `PreToolUse`, `PostToolUse`, `Stop`, `SessionStart` |
| `matcher` | Which tools the group applies to | `"Bash"`, `"Edit\|Write"` |
| `hooks` | The handlers to run | `{ "type": "command", "command": "..." }` |

Two common shapes follow.

### After an edit: format the file

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

The hook receives the tool call as JSON on standard input, so the command reads the edited file's path from it.

### Before a command: refuse it

This hook backs the `deny` rule above with a check that also catches a full path or a `bash -c` wrapper:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "jq -r '.tool_input.command' | grep -Eq 'rm +-rf' && { echo 'rm -rf is refused; delete the specific path with rm' >&2; exit 2; } || exit 0" }]
      }
    ]
  }
}
```

| Exit code | Effect |
|---|---|
| 0 | The call goes ahead |
| 2 | The call is blocked, and standard error is shown to the agent as the reason |

- Claude Code reloads settings files when they change, so an edited hook applies in the running session.
- `/hooks` shows every hook in effect and which file it came from.
- The hook is an illustration; a real one belongs in a script with tests. See [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

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

| Item | Where in Codex |
|---|---|
| Home directory | `~/.codex`, or `CODEX_HOME` if set |
| Global instructions | `AGENTS.md` (or `AGENTS.override.md`) in the home directory |
| Settings | `config.toml` in the home directory and in `<repo>/.codex/` |
| Hooks | `hooks.json` or `[hooks]` in `config.toml`, at user or repository level; non-managed hooks must be trusted |
| Sub-agent profiles | TOML files in `~/.codex/agents/` or `.codex/agents/`, inheriting the parent's sandbox and approval settings |
| Context-file size cap | `project_doc_max_bytes`, default 32 KiB |
| Extra instruction file names | `project_doc_fallback_filenames` |
| What it may do without asking | `/permissions` |
| What loaded | `/status`, `/debug-config` |

See [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}).

## Pitfalls and lessons

| Pitfall | What to do |
|---|---|
| A personal "don't ask again" lands in the local file and cannot override a project `ask` rule | Change the project rule if the team agrees |
| A deny rule misses the same program called another way | Back it with a PreToolUse hook |
| A setting written to a file you cannot write to applies for the session only | Set it in the tool that generates the file |
| A cloud session reads committed project settings only | Commit what the team relies on |
| A committed allow rule seems ignored | Check that the folder was trusted |

## Sources

- Settings files, precedence, merging, trust and troubleshooting: <https://code.claude.com/docs/en/settings.md>, read 2026-11-09.
- Default approvals, rule wildcards, compound commands, wrappers, workspace trust and modes: <https://code.claude.com/docs/en/permissions.md>, read 2026-11-09; <https://code.claude.com/docs/en/permission-modes.md>, verified 2026-11-09; `--permission-mode`: <https://code.claude.com/docs/en/cli-reference.md>, read 2026-11-09.
- Codex: <https://learn.chatgpt.com/docs/agent-configuration/agents-md>, <https://learn.chatgpt.com/docs/hooks.md>, <https://learn.chatgpt.com/docs/agent-configuration/subagents.md>, <https://learn.chatgpt.com/docs/developer-commands.md?surface=cli>, read 2026-11-09.
- Example files are illustrations written for this page.

{% include page_nav.html %}
