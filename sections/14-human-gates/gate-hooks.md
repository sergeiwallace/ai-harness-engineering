---
title: "Writing a gate hook"
parent: "Permission rules and auto mode"
grand_parent: "Human gates, decisions and alignment"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Writing a gate hook

Part of [Permission rules and auto mode]({{ '/sections/14-human-gates/permission-rules.html' | relative_url }}), in [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}).

A gate hook is a small program that runs before a tool call and decides whether the call may proceed. Use one when a permission rule's pattern cannot express the gate: when the answer depends on the whole command, the state of the repository, the time, or a lookup. This page walks through one hook from first draft to tested, and the design rules that keep a gate a gate.

## Contents

- [The contract](#the-contract)
- [A first gate](#a-first-gate)
- [Fail closed](#fail-closed)
- [Write refusals the agent can act on](#write-refusals-the-agent-can-act-on)
- [Test both arms](#test-both-arms)
- [Wiring it up](#wiring-it-up)
- [Design rules](#design-rules)
- [What a hook cannot catch](#what-a-hook-cannot-catch)
- [Pitfalls](#pitfalls)

![Sequence diagram with three alternatives. A push to main reaches Claude Code, which passes the call as JSON to the gate hook; the hook denies it with a reason and the agent is told to open a pull request. When the hook receives a malformed call it denies because it cannot check. A push to a branch gets no decision from the hook, and Claude Code runs the push in the shell.]({{ '/assets/images/14-human-gates/fail-closed-hook-sequence.svg' | relative_url }}){: style="width:100%"}

*A gate hook on three calls: it refuses a push to main, refuses when it cannot check, and stays silent on a push to a branch.*

## The contract

A PreToolUse hook receives the pending call as JSON on standard input, with fields such as `tool_name` and `tool_input`. It answers in one of two ways:

| Answer | How | Effect |
|---|---|---|
| Decide | print JSON with `hookSpecificOutput.permissionDecision` of `deny`, `ask` or `allow`, and a reason | the call is refused, prompted or allowed |
| Block | exit with code 2, with the reason on standard error | the call stops before permission rules are evaluated, even when an allow rule matches |
| Say nothing | exit 0 with no output | the normal rules and mode decide |

Deny and ask rules in settings still apply whatever the hook says ([Permission rules and auto mode]({{ '/sections/14-human-gates/permission-rules.html' | relative_url }})). The hook is layer 3 of the enforcement layers: it acts at the moment of the call. Branch protection and the test gate (layer 4) catch what it misses.

## A first gate

The gate: no push may target the main branch. A deny rule on `Bash(git push origin main)` misses `git push -f origin HEAD:main`, so this needs code.

```python
#!/usr/bin/env python3
"""Refuse any git push whose destination is the main branch."""
import json
import re
import sys

PUSH_TO_MAIN = re.compile(r"\bgit\b.*\bpush\b.*(\s|:)(refs/heads/)?main\b")

def decide(call):
    if call.get("tool_name") != "Bash":
        return None
    command = call.get("tool_input", {}).get("command", "")
    if PUSH_TO_MAIN.search(command):
        return ("deny", "Pushing to main is not allowed. Push a branch and open a pull request.")
    return None

def main():
    call = json.load(sys.stdin)
    verdict = decide(call)
    if verdict:
        decision, reason = verdict
        print(json.dumps({"hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": decision,
            "permissionDecisionReason": reason,
        }}))

if __name__ == "__main__":
    main()
```

## Fail closed

> When the guard fails, the dangerous action must still be unreachable.

If the hook above crashes, for example on malformed input, Python exits with code 1 and prints no decision:

| Exit | Meaning to Claude Code | Push |
|---|---|---|
| 0 with a deny decision | refuse | blocked |
| 2 | block | blocked |
| 1, from an uncaught exception | a hook error, not a decision | goes through |

The gate has failed open.

Wrap the work so that any failure becomes a refusal:

```python
def main():
    try:
        call = json.load(sys.stdin)
        verdict = decide(call)
    except Exception as exc:  # any failure refuses
        verdict = ("deny", f"Gate could not check this call ({exc.__class__.__name__}); refusing.")
    if verdict:
        decision, reason = verdict
        print(json.dumps({"hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": decision,
            "permissionDecisionReason": reason,
        }}))
```

The same rule applies to any guard in a script: write `check && act`, so that a failed check makes the action unreachable, never `check; act`.

## Write refusals the agent can act on

The agent reads the reason and decides what to do next. A bare "blocked" leads to retries and workarounds. A good reason says what was refused, why, and the allowed route.

| Weak | Strong |
|---|---|
| "Blocked." | "Pushing to main is not allowed. Push a branch and open a pull request." |
| "Not permitted." | "Deleting a branch needs proof it is merged. Run the merge check first, then delete." |

## Test both arms

A test that only checks the gate refuses the bad call proves half of it. A gate that refuses everything also passes that test. Check that it allows the good calls too.

```python
from gate_no_push_main import decide

def test_refuses_push_to_main():
    call = {"tool_name": "Bash", "tool_input": {"command": "git push -f origin HEAD:main"}}
    assert decide(call)[0] == "deny"

def test_allows_push_to_branch():
    call = {"tool_name": "Bash", "tool_input": {"command": "git push origin feature/export"}}
    assert decide(call) is None

def test_allows_branch_named_like_main():
    call = {"tool_name": "Bash", "tool_input": {"command": "git push origin maintenance"}}
    assert decide(call) is None


def test_refuses_when_input_is_malformed():
    import subprocess
    out = subprocess.run(["python3", "gate_no_push_main.py"], input="not json",
                         capture_output=True, text=True)
    assert '"permissionDecision": "deny"' in out.stdout
```

| Test | Arm | What a failure would mean |
|---|---|---|
| refuses push to main | refusal | the gate does not stop the dangerous call |
| allows push to a branch | allow | the gate blocks correct work and will be worked around |
| allows `maintenance` | allow | the pattern is too loose |
| refuses malformed input | fail closed | a crash lets the call through |

You can also run the hook by hand:

```bash
echo '{"tool_name":"Bash","tool_input":{"command":"git push origin main"}}' | python3 gate_no_push_main.py
```

## Wiring it up

Register the hook in the project's settings, matched to the Bash tool:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python3 \"$CLAUDE_PROJECT_DIR\"/tools/gate_no_push_main.py" }
        ]
      }
    ]
  }
}
```

Commit the hook and the settings together, so every session in the repository gets the gate.

## Design rules

1. Fail closed: any error refuses.
2. Be fast: the hook runs before every matching call. Keep it well under a second, and set a timeout.
3. Be specific: refuse the dangerous case and allow everything else; a gate that blocks correct work gets worked around.
4. Name the allowed route in every refusal.
5. Test both arms: one test that it refuses, at least one that it allows.
6. Log refusals to a ledger, so you can see how often the gate fires and whether it blocks correct work ([Logging and observability]({{ '/sections/16-logging-and-evaluation/logging-and-observability.html' | relative_url }})).

## What a hook cannot catch

A hook reads the command text of one tool call. Some routes to the same action never show it that text:

| Route | Example | The layer that catches it |
|---|---|---|
| A script that pushes | `./release.sh`, which runs `git push origin main` inside it | branch protection on the server |
| An alias or a variable | `git p`, or `git push origin "$TARGET"` | branch protection on the server |
| Another tool | a merge done through the hosting service's API | the service's own rules and required reviews |
| Hooks turned off | a run started with `--settings '{"disableAllHooks": true}'` | permission deny rules, which still apply, and branch protection |

A hook is one enforcement mechanism among several. Permission deny rules also block, a hook can refuse through its JSON `permissionDecision` as well as through exit code 2, and hooks can be switched off for a run. This is why the hook sits in front of layer 4, not instead of it.

## Examples from my own harness

- Many of my guards started as written rules that agents broke; each became a hook that refuses the action and names the route to take instead.
- Guards are written so that a failed check makes the action unreachable, and each has tests for both the refusal and the allowed case.

To start: take the one rule in your context file that agents break most often, and write it as a hook with the tests above.

## Pitfalls

- A hook that prints a decision and also exits non-zero sends mixed signals. Print the JSON decision and exit 0, or exit 2 with the reason on standard error, not both.
- A slow hook slows every matching call. A hook that calls the network on each Bash command adds that latency to every command.
- A regex written against one spelling of a command misses the others. Collect real commands from your logs and add each new spelling as a test.

## Sources

- PreToolUse hooks, their precedence with permission rules, exit code 2 blocking before rules are evaluated, and `disableAllHooks` turning hooks off for a run: <https://code.claude.com/docs/en/permissions.md>, verified 2026-11-09.
- Hook input and output fields: <https://code.claude.com/docs/en/hooks.md>, not yet verified against the primary source for this page.
- The guard design rules: from the author's own harness, described conceptually.

{% include page_nav.html %}
