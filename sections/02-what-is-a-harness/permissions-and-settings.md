---
title: "Permissions and settings"
parent: "What is a harness?"
nav_order: 5
redirect_from:
  - /sections/02-ai-harnesses/permissions-and-settings.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Permissions and settings

Settings configure the harness itself.

- Permissions are the part of settings that decide which actions the agent may take without asking.
- Claude Code's docs put it plainly: "Permission rules are enforced by Claude Code, not by the model."
- Permission rules are the first hard control in the [systems approach]({{ '/sections/02-what-is-a-harness/' | relative_url }}#a-systems-approach-to-harness-engineering), and the allow list decides how often you are interrupted.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Settings files

### Scopes

| Scope | File | Applies to | Typical contents |
|---|---|---|---|
| Managed | `managed-settings.json`, an MDM policy, or server-managed settings | everyone the organisation deploys it to | security policy and compliance limits |
| User | `~/.claude/settings.json` | you, every project on this machine | your hooks, default model, personal allow rules |
| Shared project | `.claude/settings.json` | everyone in the project, once committed | project hooks, shared allow and deny rules, plugins |
| Project local | `.claude/settings.local.json` | you, this project | personal overrides; your "don't ask again" approvals |

- Claude Code writes `.claude/settings.local.json` itself when you choose "Yes, and don't ask again" on a prompt, and adds it to your global git excludes so it stays out of commits.
- Settings files are strict JSON: a comment or a trailing comma is an error. Add `"$schema": "https://json.schemastore.org/claude-code-settings.json"` for autocomplete in your editor.
- Claude Code reloads settings when they change, so most edits, including `permissions` and `hooks`, reach a running session without a restart.

### Precedence

For an ordinary key, the highest level that sets it wins.

1. Managed settings.
2. The command line: `claude --settings '<json or file>'`, for one session.
3. Project local.
4. Shared project.
5. User.

Lists such as `permissions.allow` merge across all levels; no level removes another level's entries.

| Example | Result |
|---|---|
| user allows `Bash(git push *)`, project asks for it | ask: ask rules beat allow rules |
| project allows `Read(./.env)`, user denies it | deny: deny rules beat everything |
| local sets `"model": "claude-sonnet-5"`, shared project sets an Opus model | Sonnet, for you, in this project |
| managed settings set a key | that value, whatever your own files say |

### Checking what loaded

| Command | Shows |
|---|---|
| `/status` | which settings files loaded (the Setting sources line), including any managed source |
| `/permissions` | the rules by scope; add or remove them |
| `claude doctor` | entries Claude Code rejected, and why |

### What else lives in settings

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "claude-opus-5-5",
  "env": { "PYTHONDONTWRITEBYTECODE": "1" },
  "permissions": { "allow": [], "ask": [], "deny": [], "defaultMode": "default" },
  "hooks": {},
  "enabledPlugins": {}
}
```

## Permission rules

![A vertical flow chart of one tool call, for example git push, through the permission checks. PreToolUse hooks run first, before any rule; a deny there refuses the call and the hook says why. With no decision from the hooks, the rules are checked in order from every scope. If a deny rule matches, the call is refused and never runs. If an ask rule matches, such as Bash(git push \*), the person is prompted and the call runs if they approve. If an allow rule matches, such as tests, linters or git status, it runs with no prompt. If no rule matches, the permission mode set by defaultMode decides.]({{ '/assets/images/diagrams/02-ai-harnesses/permission-path-flow.svg' | relative_url }})
*Figure: one tool call, checked by hooks first and rules second: deny, then ask, then allow. Made with archify.*

*One push, checked by hooks first and rules second. The diagram writes the rule as `Bash(git push:*)`, an equivalent form of `Bash(git push *)`. Our own diagram, made with archify.*

### The three lists

| List | Effect | Use for |
|---|---|---|
| `allow` | runs without a prompt | safe, frequent actions: tests, linters, `git status` |
| `ask` | prompts the person | actions you want to see: pushes, installs |
| `deny` | refused | actions that must never happen: reading secrets |

Rules are checked in a fixed order: deny, then ask, then allow.

- The first match in that order decides, however specific a later rule is.
- A deny rule that names a tool with no pattern, such as `"WebFetch"`, removes the tool from the model's context entirely.

### Rule syntax

| Rule | Matches |
|---|---|
| `Bash(npm run test *)` | `npm run test` and anything after it, as a prefix |
| `Bash(git status)` | exactly `git status` |
| `Bash(ls *)` | `ls` and `ls -la`, but not `lsof`: the space before `*` is part of the rule |
| `Read(./.env)` | the `.env` file in the current folder |
| `Read(**/.env)` | any `.env` at or under the current folder |
| `Read(~/Documents/*.pdf)` | PDFs in your home Documents folder |
| `Read(//etc/**)` | an absolute path from the filesystem root |
| `Edit(/src/**/*.ts)` | TypeScript under `src/`, relative to the settings file's project |
| `WebFetch(domain:docs.example.com)` | fetches from one domain |
| `mcp__notes__send`, `mcp__notes__*` | one MCP tool, or every tool from one server |

- `Read` and `Edit` paths use gitignore pattern syntax: `*` within one folder, `**` across folders.
- `Bash(ls:*)` is an older equivalent of `Bash(ls *)`; the prompt writes the space form.
- Put the `*` after the subcommand: `Bash(git log *)` limits the rule to `git log`.

### Compound commands

Claude Code splits a command on `&&`, `||`, `;`, `|`, `&` and newlines before it matches rules.

1. An allow rule approves a compound command only if every part is allowed: `Bash(npm test *)` does not approve `npm test && curl ...`.
2. A deny or ask rule applies if any part matches, including a part inside a subshell or a `for` loop.
3. A rule matches the command text as written. `Bash(curl *)` does not match `/usr/bin/curl ...` or `sh -c 'curl ...'`.

> For limits that must hold against every spelling, match on the effect: the sandbox for files and network, or a [guard hook]({{ '/sections/02-what-is-a-harness/guard-hooks.html' | relative_url }}) that parses the command. A pattern that tries to constrain arguments, such as `Bash(curl http://github.com/ *)`, misses `curl -X GET http://github.com/...` and redirects.
{: .warning }

### A worked example

```json
{
  "permissions": {
    "allow": [
      "Bash(uv run pytest *)",
      "Bash(uv run ruff *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)"
    ],
    "ask": ["Bash(git push *)", "Bash(uv add *)"],
    "deny": ["Read(./.env)", "Read(./.env.*)", "Read(./secrets/**)"]
  }
}
```

1. The tests, the linter and read-only git commands run unasked, which removes most prompts in a working afternoon.
2. Adding a dependency and pushing wait for you.
3. The secrets cannot be read through the `Read` tool.
4. A `cat .env` through `Bash` is a separate route; the sandbox or a `Bash` deny rule closes it.

### Project rules and workspace trust

A cloned repository's `.claude/settings.json` could allow anything, so Claude Code holds part of it back.

| From the shared project file | Applies |
|---|---|
| `deny` and `ask` rules | at once |
| `allow` rules, extra directories, most `env` values, marketplaces | after you trust the folder |
| `defaultMode: auto` or `bypassPermissions` | never; set them in user or managed settings |

### Hooks and rules together

`PreToolUse` hooks run before the permission prompt.

| Hook returns | A deny rule matches | An ask rule matches | No rule matches |
|---|---|---|---|
| deny | refused | refused | refused |
| allow | refused | prompts | runs |
| ask | refused | prompts | prompts |
| no decision | refused | prompts | the normal default for that tool |

A hook can always refuse, and it can never override a deny or ask rule. See [Harness hooks]({{ '/sections/02-what-is-a-harness/hooks.html' | relative_url }}).

## Permission modes

| Mode | Behaviour | When |
|---|---|---|
| `default` (shown as Manual) | prompts on first use of each tool that needs approval | normal work |
| `acceptEdits` | accepts file edits and common file commands (`mkdir`, `touch`, `mv`, `cp`) inside the working folders | a trusted editing task |
| `plan` | reads and runs read-only commands; does not edit source files | before a large change |
| `auto` | no routine prompts; a background classifier checks shell commands and network requests against your request | long tasks you trust, where available |
| `dontAsk` | denies every call that would have prompted | unattended runs with a tight allow list |
| `bypassPermissions` | skips prompts, except for a short list of actions no mode approves | only in an isolated container or VM |

- Set the default with `permissions.defaultMode`, or for one session with `--permission-mode`.
- An organisation can forbid the riskiest modes with `permissions.disableBypassPermissionsMode` and `permissions.disableAutoMode` set to `"disable"`.

Codex has the same idea in two settings:

| Codex setting | Values |
|---|---|
| `approval_policy` | `on-request`, `never`, or a granular table |
| `sandbox_mode` | `read-only`, `workspace-write`, `danger-full-access` |

<!-- IMAGE NEEDED: a screenshot of a Claude Code permission prompt for a Bash command, with its Yes / Yes, and don't ask again / No options; search: claude code permission prompt screenshot, claude code bash approval dialog -->

## Sandboxing

A sandbox limits what a command can reach at the operating-system level. It holds even when a rule or hook misses a spelling, because it acts on what the command touches, whatever its text.

| Limit | Typical setting |
|---|---|
| write access | the project folder and a temporary folder only |
| read access | the project and the tools it needs; not your home folder's secrets |
| network | off, or a list of allowed hosts |

Claude Code turns it on with `sandbox.enabled`; an organisation can enforce it from managed settings. Use the three controls together:

1. A sandbox for the broad limit.
2. Permission rules for the everyday prompts.
3. Hooks for the cases a pattern cannot express.

<!-- IMAGE NEEDED: three nested rings around one agent: sandbox (files and network), permission rules (per tool call), hooks (parsed checks); search: agent sandbox layers diagram, defense in depth AI agent permissions -->

## Pitfalls

| Pitfall | Result | Fix |
|---|---|---|
| "Yes, and don't ask again" on a broad prefix | a rule wider than you meant, saved to your local file | edit the rule in `/permissions` to the narrow form |
| an allow rule in the shared project file | it waits for workspace trust, so teammates still see prompts | expected; trust the folder |
| a deny rule on `Read(./.env)` only | `cat .env` through `Bash` still works | add a sandbox read limit or a `Bash` rule |
| `bypassPermissions` on your own machine | every action runs, including deletes and pushes | use it only inside a disposable container |
| a broken settings file | Claude Code skips it or the broken entry | read the Settings Error or Warning; run `claude doctor` |

## Settings in my harness

My harness keeps settings in source and installs them, like its [compiled context files]({{ '/sections/02-what-is-a-harness/compiling-context-files.html' | relative_url }}).

- Every machine gets the same hooks and shared permission rules from one source.
- Machine-specific differences live in that machine's overlay: which tools exist, which actions need approval there.
- Approval is policy: a command that would work is still held for a person when it reaches an outside service.
- After an install, `/status`, `/hooks` and `/permissions` in a live session confirm what arrived.

## Related pages

- [Harness hooks]({{ '/sections/02-what-is-a-harness/hooks.html' | relative_url }}): hooks are declared in the same files.
- [Native commands and tools]({{ '/sections/02-what-is-a-harness/native-commands-and-tools.html' | relative_url }}): the tool names rules match.
- [Human gates]({{ '/sections/14-human-gates/' | relative_url }}): which actions should wait for a person.

## Sources

- [Claude Code settings and precedence](https://code.claude.com/docs/en/settings.md), verified 2026-11-09.
- [Claude Code permissions, rule syntax and modes](https://code.claude.com/docs/en/permissions.md), verified 2026-11-09.
- [Codex `approval_policy` and `sandbox_mode`](https://learn.chatgpt.com/docs/config-file/config-reference), verified 2026-11-09.
- The settings install and overlays are from the author's own harness, described conceptually.

{% include page_nav.html %}
