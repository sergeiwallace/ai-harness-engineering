---
title: "Permission rules and auto mode"
parent: "Human gates, decisions and alignment"
nav_order: 3
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Permission rules and auto mode

As of 2026-11-09.

Part of [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}). Sub-page: [Writing a gate hook]({{ '/sections/14-human-gates/gate-hooks.html' | relative_url }}).

Permission rules are the gates the tool enforces, as opposed to the gates an agent is asked to respect. Claude Code's documentation says: "Permission rules are enforced by Claude Code, not by the model. Instructions in your prompt or `CLAUDE.md` shape what Claude tries to do, but they don't change what Claude Code allows." This page covers Claude Code's rules, modes and hooks, with a short note on Codex.

## Contents

- [Rules: allow, ask and deny](#rules-allow-ask-and-deny)
- [Rule syntax](#rule-syntax)
- [Permission modes](#permission-modes)
- [Hooks as programmable gates](#hooks-as-programmable-gates)
- [Codex approvals and sandbox](#codex-approvals-and-sandbox)
- [Worked configurations](#worked-configurations)
- [Pitfalls](#pitfalls)
- [Sources](#sources)

## Rules: allow, ask and deny

Rules live in the `permissions` block of a settings file (user, project or local) and can be edited with the `/permissions` command. Each rule matches a tool call:

- `deny` refuses the call.
- `ask` prompts the person, even in a mode that would otherwise run it.
- `allow` runs it without a prompt.

The order is fixed. From the documentation: "Rules are evaluated in order: deny, then ask, then allow. The first match in that order determines the outcome, and rule specificity doesn't change the order." An allow rule cannot carve an exception out of a deny rule: a deny on `Bash(aws *)` blocks `aws s3 ls` even if `Bash(aws s3 ls)` is allowed.

![Sequence diagram with two alternatives. A push to main reaches the PreToolUse hook, which denies it and tells the agent to open a pull request. A push to a branch passes the hook with no decision, matches an ask rule, the person is prompted and says yes once, and the push runs.]({{ '/assets/images/14-human-gates/permission-layers-sequence.svg' | relative_url }}){: style="width:100%"}

*One push, through the permission layers: the hook refuses a push to main, and an ask rule prompts for a push to a branch.*

![A tool call meets deny rules first, which refuse it on a match; then a pre-tool hook, which can refuse it or force a prompt; then ask rules, which prompt the person; then allow rules, which let it run; and otherwise the permission mode decides, where auto mode has a classifier approve routine actions.]({{ '/assets/images/14-human-gates/permission-layers.svg' | relative_url }})

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

Two limits from the documentation are worth knowing. A Bash deny rule matches the command text, so it does not match the same program called by full path or inside `sh -c`; for a restriction that must hold, pair it with the sandbox. And `Edit` rules cover every built-in tool that edits files, so `Edit(docs/**)` is the rule to use rather than `Write(docs/**)`.

## Permission modes

The mode decides what happens to a call no rule matched. The documented modes are:

| Mode | Behaviour |
|---|---|
| `default` (Manual) | prompts on first use of each tool |
| `acceptEdits` | accepts file edits and common filesystem commands in the working directories |
| `plan` | reads and explores but does not edit source files |
| `auto` | runs without routine prompts; "a background classifier checks that they align with your request" before actions such as shell commands and network requests |
| `dontAsk` | auto-denies every call that would otherwise prompt |
| `bypassPermissions` | skips prompts, except for actions no mode auto-approves |

Auto mode is the one built for long runs. Instead of the person approving each command, a classifier reviews each action against the request. The person's deny and ask rules still apply, so the classifier works inside the gates you set.

> In auto mode your deny and ask rules still apply; the classifier handles the calls they do not match.

### Example: choosing a mode for a long run

#### The comparison

| Mode | Overnight refactor outcome |
|---|---|
| Manual | stalls at the first command until morning |
| `bypassPermissions` | runs, and nothing stops a push to main |
| Auto plus rules | runs, cannot push to main, leaves the install question waiting |

#### The setup

A person starts an overnight refactor. Manual mode would stall at the first command. `bypassPermissions` would remove every prompt, including the ones the person cares about. Auto mode with a deny rule on pushes to the main branch and an ask rule on package installs lets the refactor run, stops it from pushing to main, and leaves the install question waiting in the morning.

## Hooks as programmable gates

A PreToolUse hook is your own program, run before each tool call. It receives the call as JSON on standard input and can deny it, force a prompt, or let it proceed. Use a hook when a rule's pattern is not enough: when the decision depends on the full command, on the state of the repository, or on a lookup.

Hooks do not override rules. The documentation states that "a matching deny rule blocks the call, and a matching ask rule still prompts even when the hook returned `"allow"` or `"ask"`."

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

The reason string matters as much as the refusal: the agent reads it and changes course, so a useful reason names the allowed route. [Writing a gate hook]({{ '/sections/14-human-gates/gate-hooks.html' | relative_url }}) takes this hook from first draft to tested, including why it must fail closed. [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers hooks in general.

## Codex approvals and sandbox

Codex has the same two layers under different names: an approval policy that decides when it asks, and a sandbox mode that limits what commands can touch. Both are set in `config.toml` or per run. One pairing for local work is a workspace-write sandbox with approval on request, so Codex can edit the project and asks before anything outside it.

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

Tests, linting and read-only git run freely. Pushes and installs ask. Secrets cannot be read, and network tools are denied in favour of the WebFetch tool, which has its own domain rules.

### A long autonomous run

Same rules, auto mode, and one hook that refuses pushes to the main branch with a message pointing at the pull request route. The person reviews the pull request in the morning.

## Examples from my own harness

- Deny rules and hooks guard the actions on my never-alone list, and each refusal message names the allowed route, so an agent that hits one changes course instead of retrying.
- Gates are set per repository class: in my own repositories push and merge are ungated once the local test gate is green; in shared repositories every gate stands.

To start: put your never-alone list into `ask` and `deny` rules in the project's `.claude/settings.json`, and commit it so every session gets them.

## Pitfalls

- A broad allow rule granted to stop prompts ("Bash(*)") removes every gate the prompts were providing.
- Bash patterns match text. An agent that cannot run `curl` may still reach the network another way, so use the sandbox where a restriction must hold.
- A hook with a vague refusal ("blocked") leads to retries. Say why, and what to do instead.

## Sources

- Configure permissions, rule order, rule syntax, permission modes and hook precedence: <https://code.claude.com/docs/en/permissions.md>, verified 2026-11-09.
- Codex approval policy and sandbox mode: not yet verified against the primary source for this page.

{% include page_nav.html %}
