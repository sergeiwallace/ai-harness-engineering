---
title: "Permission rules and auto mode"
parent: "Human gates, decisions and alignment"
nav_order: 3
has_children: true
has_toc: false
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Permission rules and auto mode

Part of [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}). Sub-page: [Writing a gate hook]({{ '/sections/14-human-gates/gate-hooks.html' | relative_url }}).

Permission rules are the gates the tool enforces, as opposed to the gates an agent is asked to respect. Claude Code's documentation says: "Permission rules are enforced by Claude Code, not by the model. Instructions in your prompt or `CLAUDE.md` shape what Claude tries to do, but they don't change what Claude Code allows." This page covers Claude Code's rules, modes and hooks, with a short note on Codex.

## Contents

- [Where these sit in the enforcement layers](#where-these-sit-in-the-enforcement-layers)
- [Where rules live](#where-rules-live)
- [Rules: allow, ask and deny](#rules-allow-ask-and-deny)
- [Rule syntax](#rule-syntax)
- [Permission modes](#permission-modes)
- [Hooks as programmable gates](#hooks-as-programmable-gates)
- [Codex approvals and sandbox](#codex-approvals-and-sandbox)
- [Worked configurations](#worked-configurations)
- [Pitfalls](#pitfalls)
- [Sources](#sources)

## Where these sit in the enforcement layers

A systems approach puts each gate into the system, in layers:

1. Instructions in context files, skills and prompts: "ask before you push". The second line of defence.
2. Workflows and tools that build the step in: a merge skill that always opens a pull request.
3. Hooks, scripts and linters at the moment of action: PreToolUse hooks, deny and ask rules, the sandbox.
4. Validation and tests before a change lands: branch protection, the test gate, review.

Layers 2 to 4 do the enforcing. This page is about layer 3 as Claude Code and Codex provide it. When an agent crosses a gate, the fix goes into one of layers 2 to 4; a new context-file line is at most a second layer.

## Where rules live

Rules sit in the `permissions` block of a settings file, and `/permissions` edits them in a session.

| File | Scope | Typical use |
|---|---|---|
| Managed settings, deployed by an administrator | every user on the machine or organisation | policy that nobody may override |
| `~/.claude/settings.json` | you, in every project | your personal deny list |
| `.claude/settings.json`, committed | everyone working in the repository | the project's shared gates |
| `.claude/settings.local.json`, not committed | you, in this repository | your own allows; "Yes, and don't ask again" saves here |

How the files combine:

- Managed settings are highest; no other level, including command-line flags, can override a managed rule.
- A deny at any level wins: a user-level deny blocks a project-level allow, and the reverse.
- Command-line flags such as `--allowedTools` and `--disallowedTools` add to the rules for one session.

## Rules: allow, ask and deny

Each rule matches a tool call:

- `deny` refuses the call.
- `ask` prompts the person, even in a mode that would otherwise run it.
- `allow` runs it without a prompt.

The order is fixed. From the documentation: "Rules are evaluated in order: deny, then ask, then allow. The first match in that order determines the outcome, and rule specificity doesn't change the order."

| Rules present | Call | Result |
|---|---|---|
| deny `Bash(aws *)`, allow `Bash(aws s3 ls)` | `aws s3 ls` | refused: an allow cannot carve an exception out of a deny |
| ask `Bash(git push *)`, allow `Bash(git push origin feature/*)` | `git push origin feature/x` | prompts: ask is checked before allow |
| allow `Bash(npm run *)` | `npm run test` | runs |

![Sequence diagram with two alternatives. A push to main reaches the PreToolUse hook, which denies it and tells the agent to open a pull request. A push to a branch passes the hook with no decision, matches an ask rule, the person is prompted and says yes once, and the push runs.]({{ '/assets/images/14-human-gates/permission-layers-sequence.svg' | relative_url }}){: style="width:100%"}

*One push, through the permission layers: the hook refuses a push to main, and an ask rule prompts for a push to a branch.*

![A tool call meets deny rules, which refuse it on a match; a pre-tool hook, which can refuse it or force a prompt; ask rules, which prompt the person; allow rules, which let it run; and otherwise the permission mode decides, where auto mode has a classifier approve routine actions.]({{ '/assets/images/14-human-gates/permission-layers.svg' | relative_url }}){: style="width:100%"}

*The layers as a decision path. Deny and ask rules hold whatever the hook returns, and a hook that blocks stops the call even when an allow rule matches.*

<!-- IMAGE NEEDED: a screenshot of the Claude Code /permissions screen listing allow, ask and deny rules; search: "Claude Code /permissions", "Claude Code permission rules settings" -->

## Rule syntax

A rule is a tool name, optionally followed by a pattern in parentheses.

| Rule | Matches |
|---|---|
| `Bash(npm run *)` | any command starting with `npm run` |
| `Bash(git push *)` | any push |
| `Read(./.env)` | reading `.env` in the current directory |
| `Read(./secrets/**)` | reading anything under `secrets/` |
| `Edit(/docs/**)` | edits under `docs/` in the project |
| `WebFetch(domain:example.com)` | fetches from example.com |

Limits from the documentation:

- A Bash rule matches the command text. A deny rule does not match the same program called by full path or inside `sh -c`. For a restriction that must hold, pair it with the sandbox or a hook.
- `Edit` rules cover every built-in tool that edits files, so write `Edit(docs/**)` rather than `Write(docs/**)`.
- `Read` deny rules also apply, on a best-effort basis, to Grep, Glob and `@file` mentions.
- A `.claudeignore` file has no effect; move its entries into `Read` deny rules.

## Permission modes

The mode decides what happens to a call no rule matched.

| Mode | Behaviour |
|---|---|
| `default` (Manual) | prompts on first use of each tool |
| `acceptEdits` | accepts file edits and common filesystem commands in the working directories |
| `plan` | reads and explores but does not edit source files |
| `auto` | runs without routine prompts; "a background classifier checks that they align with your request" before actions such as shell commands and network requests |
| `dontAsk` | auto-denies every call that would otherwise prompt |
| `bypassPermissions` | skips prompts, except for actions no mode auto-approves |

Setting and locking modes:

```json
{
  "permissions": {
    "defaultMode": "acceptEdits",
    "disableBypassPermissionsMode": "disable",
    "disableAutoMode": "disable"
  }
}
```

- `defaultMode` sets the mode new sessions start in.
- `disableBypassPermissionsMode` and `disableAutoMode` stop those modes from being used. They are most useful in managed settings, where nobody can override them.

> In auto mode your deny and ask rules still apply; the classifier handles the calls they do not match.

### Example: choosing a mode for a long run

A person starts an overnight refactor.

| Mode | Overnight outcome |
|---|---|
| Manual | stalls at the first command until morning |
| `bypassPermissions` | runs, and nothing stops a push to main |
| `auto` plus rules | runs, cannot push to main, and leaves the package-install question waiting |

The rules for the third row: a deny on pushes to main and an ask on `npm install`.

## Hooks as programmable gates

A PreToolUse hook is your own program, run before the permission prompt for each tool call. It receives the call as JSON on standard input and can deny it, force a prompt, or let it proceed. Use a hook when a rule's pattern is not enough:

- the decision depends on the full command, such as any push whose target is main, in any spelling;
- it depends on the state of the repository, such as uncommitted changes;
- it depends on a lookup, such as an allowlist of hosts.

How hooks and rules combine, from the documentation:

| Hook result | Matching rule | Outcome |
|---|---|---|
| allow or ask | deny | refused |
| allow | ask | prompts |
| blocks (exit code 2) | allow | refused: the block stops the call before rules are evaluated |
| no decision | none | the mode decides |

A first draft:

```python
#!/usr/bin/env python3
"""PreToolUse hook: refuse a push to main, explain the alternative."""
import json
import sys

call = json.load(sys.stdin)
command = call.get("tool_input", {}).get("command", "")
if call.get("tool_name") == "Bash" and "git push" in command and " main" in command:
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": "Push a branch and open a pull request instead of pushing to main.",
        }
    }))
sys.exit(0)
```

This draft misses `git push origin HEAD:main` and lets the call through if it crashes. [Writing a gate hook]({{ '/sections/14-human-gates/gate-hooks.html' | relative_url }}) fixes both and adds tests. The reason string matters as much as the refusal: the agent reads it and changes course, so a useful reason names the allowed route. [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers hooks in general.

## Codex approvals and sandbox

Codex has the same two ideas under different names (not yet verified against the Codex docs for this page):

| Setting | Decides | Example value |
|---|---|---|
| `approval_policy` | when Codex asks before acting | `on-request` |
| `sandbox_mode` | what commands can touch | `workspace-write` |

Both are set in `config.toml` or per run. One pairing for local work is a workspace-write sandbox with approval on request, so Codex can edit the project and asks before anything outside it.

```toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

## Worked configurations

### A cautious project default

```json
{
  "permissions": {
    "allow": ["Bash(npm run test *)", "Bash(npm run lint *)", "Bash(git status)", "Bash(git diff *)"],
    "ask":   ["Bash(git push *)", "Bash(npm install *)"],
    "deny":  ["Read(./.env)", "Read(./secrets/**)", "Bash(curl *)", "Bash(wget *)"]
  }
}
```

| Group | Effect |
|---|---|
| allow | tests, linting and read-only git run freely |
| ask | pushes and installs prompt |
| deny | secrets cannot be read; `curl` and `wget` are refused in favour of the WebFetch tool, which has its own domain rules |

### A long autonomous run

| Layer | Setting |
|---|---|
| Rules | the cautious project default above |
| Mode | `auto`, so routine commands run without a prompt |
| Hook | refuses any push to main, naming the pull request route ([Writing a gate hook]({{ '/sections/14-human-gates/gate-hooks.html' | relative_url }})) |
| Validation | branch protection on main, and the test gate before merge |

The person reviews the pull request in the morning.

## Examples from my own harness

- Deny rules and hooks guard the actions on my never-alone list, and each refusal message names the allowed route, so an agent that hits one changes course instead of retrying.
- Gates are set per repository class: in my own repositories push and merge are ungated once the local test gate is green; in shared repositories every gate stands.

To start: put your never-alone list into `ask` and `deny` rules in the project's `.claude/settings.json`, and commit it so every session gets them.

## Pitfalls

- A broad allow rule granted to stop prompts (`Bash(*)`) removes every gate the prompts were providing.
- Bash patterns match text. An agent that cannot run `curl` may still reach the network another way, so use the sandbox where a restriction must hold.
- Clicking "Yes, and don't ask again" adds a permanent allow rule to `.claude/settings.local.json`. Review that file now and then.
- A hook with a vague refusal ("blocked") leads to retries. Say why, and what to do instead.

## Sources

- Configure permissions: settings locations and precedence, rule order and syntax, permission modes, `defaultMode`, the mode-disabling keys, and how hooks interact with rules: <https://code.claude.com/docs/en/permissions.md>, verified 2026-11-09.
- Codex approval policy and sandbox mode: not yet verified against the primary source for this page.

{% include page_nav.html %}
