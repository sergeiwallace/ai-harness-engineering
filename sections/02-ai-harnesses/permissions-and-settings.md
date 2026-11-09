---
title: "Permissions and settings"
parent: "AI harnesses"
nav_order: 5
---

{% include page_nav.html %}

# Permissions and settings

As of 2026-11-09. Part of [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}).

Settings configure the harness itself. Permissions are the part of settings that decide which actions the agent may take without asking. They are the first hard control: a denied action is refused whatever the model decides, and the allow list decides how often you are interrupted.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Settings files

### Scopes

| Scope | File | Shared | Typical contents |
|---|---|---|---|
| Managed policy | set by an administrator | every user on the machine | organisation limits; cannot be overridden |
| User | `~/.claude/settings.json` | you, every project | your hooks, your default model, personal allow rules |
| Project | `.claude/settings.json` | committed, the team | project hooks, shared allow and deny rules |
| Local | `.claude/settings.local.json` | you, this project, not committed | experiments, machine-specific paths |

### Precedence

When scopes disagree, the harness resolves them in a fixed order.

1. Managed policy wins over everything and cannot be overridden.
2. For ordinary settings, the more specific scope wins: local over project over user.
3. Permission lists are merged across all scopes, not replaced.
4. Within the merged lists, deny beats ask, and ask beats allow.

| Example | Result |
|---|---|
| user allows `Bash(git push:*)`, project asks for it | ask |
| project allows `Read(./.env)`, user denies it | deny |
| local sets `model: sonnet`, user sets `model: opus` | sonnet in this project |

### What else lives in settings

```json
{
  "model": "opus",
  "env": { "PYTHONDONTWRITEBYTECODE": "1" },
  "permissions": { "allow": [], "ask": [], "deny": [], "defaultMode": "default" },
  "hooks": {},
  "enabledPlugins": {}
}
```

## Permission rules

![A sequence diagram of one tool call through the permission checks. The model asks the harness to run git push. The harness sends the event to the PreToolUse hooks first; a deny there ends the call, and here the hook exits 0 with no decision. The harness then matches the permission rules from every scope, where deny beats ask beats allow, and the rule Bash(git push:*) says ask. The harness prompts the person, who approves. The harness runs the command and returns the output to the model as the tool result.]({{ '/assets/images/02-ai-harnesses/permission-decision.png' | relative_url }})

*One push, checked by hooks first and rules second. Our own diagram, made with archify.*

### The three lists

| List | Effect | Use for |
|---|---|---|
| `allow` | runs without a prompt | safe, frequent actions: tests, reads, `git status` |
| `ask` | prompts the person | actions you want to see: pushes, installs |
| `deny` | refused | actions that must never happen: reading secrets |

### Rule syntax

| Rule | Matches |
|---|---|
| `Bash(npm run test:*)` | any command starting `npm run test` |
| `Bash(git diff:*)` | `git diff` with any arguments |
| `Read(./.env)` | reading the `.env` file in the project |
| `Read(./secrets/**)` | anything under `secrets/` |
| `Edit(src/**)` | edits under `src/` |
| `WebFetch(domain:docs.example.com)` | fetching from one domain |
| `mcp__notes__send` | one MCP tool |

### A worked example

```json
{
  "permissions": {
    "allow": [
      "Bash(uv run pytest:*)",
      "Bash(uv run ruff:*)",
      "Bash(git status)",
      "Bash(git diff:*)",
      "Bash(git log:*)"
    ],
    "ask": ["Bash(git push:*)", "Bash(uv add:*)"],
    "deny": ["Read(./.env)", "Read(./.env.*)", "Read(./secrets/**)"]
  }
}
```

The tests, the linter and read-only git commands run unasked, which removes most prompts in a working afternoon. Adding a dependency and pushing wait for you. The secrets are unreachable.

> A prefix rule on Bash matches the start of the command string. `Bash(git diff:*)` does not stop `git diff; curl ...` from being judged by the rest of the command, and harness versions differ in how they treat compound commands. For rules that must hold against every spelling, use a [guard hook]({{ '/sections/02-ai-harnesses/guard-hooks.html' | relative_url }}).
{: .warning }

## Permission modes

| Mode | Behaviour | When |
|---|---|---|
| `default` | asks for anything not allowed | normal work |
| `acceptEdits` | file edits run unasked; shell still asks | a trusted editing task |
| `plan` | read-only; the agent proposes a plan | before a large change |
| `bypassPermissions` | nothing asks | only in an isolated sandbox or container |

Codex has the same idea in two settings: an approval policy (when to ask) and a sandbox mode (what the shell may touch: read-only, the workspace, or full access).

## Sandboxing

A sandbox limits what a command can reach at the operating-system level. It holds even when a rule or hook misses a spelling.

| Limit | Typical setting |
|---|---|
| Write access | the project folder and a temporary folder only |
| Read access | the project and the tools it needs; not your home folder's secrets |
| Network | off, or a list of allowed hosts |

Use the three together:

1. A sandbox for the broad limit.
2. Permission rules for the everyday prompts.
3. Hooks for the cases a pattern cannot express.

## Settings in my harness

My harness keeps settings in source and installs them, like its [compiled context files]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}).

- Every machine gets the same hooks and shared permission rules from one source.
- Machine-specific differences live in that machine's overlay: which tools exist, which actions need approval there.
- Approval is policy: a command that would work is still held for a person when it reaches an outside service.
- The installer reports what it changed, and `/hooks` and `/permissions` in a live session confirm it arrived.

## Related pages

- [Harness hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}): hooks are declared in the same files.
- [Human gates]({{ '/sections/14-human-gates/' | relative_url }}): which actions should wait for a person.

## Sources

- Claude Code settings and permissions: <https://code.claude.com/docs/en/settings.md>, <https://code.claude.com/docs/en/iam.md>.
- Codex approvals and sandbox: <https://developers.openai.com/codex/security>.
- The settings install and overlays are from the author's own harness, described conceptually.

{% include page_nav.html %}
