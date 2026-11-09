---
title: "Writing a guard"
parent: "Hooks"
grand_parent: "Hooks, scripts and linters"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Writing a guard

A guard is a [hook]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}) that refuses an action. It can enforce how work should be done, such as refusing a commit without the required trailers, or stop a mistake that has happened, such as a pattern kill. Whether agents can work with a guard depends on:

1. what its refusal says;
2. what it does with commands it cannot read;
3. whether the risky step is unreachable when the check fails.

This page covers each, then the stories of five guards in my harness that started as written rules agents broke. It is a sub-topic of [Hooks]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## A refusal and a recovery

![Sequence diagram with four lanes: Model, Client, PreToolUse hook and Shell. The model asks for pkill -f pytest; the hook exits 2 with the message that pattern kills are refused; the client returns the block to the model. The model runs pgrep -a pytest, which the hook allows, and gets process 4121. The model runs kill 4121; the hook allows one PID; only that process stops.]({{ '/assets/images/06-hooks-scripts-and-linters/guard-refusal-sequence.png' | relative_url }})
*The refusal message in the red arrow names the allowed form, and the agent's next two calls use it.*

## Writing one, step by step

1. Name the action to refuse, as a command or file pattern a program can match: `pkill -f`, `git push --force`, an edit under `migrations/`.
2. Pick the event and matcher: `PreToolUse` with `Bash` for shell commands, `Edit|Write` for file changes.
3. Read the event JSON from standard input with `jq` and take the field you need, such as `.tool_input.command`.
4. Decide in three outcomes:
   - Safe and readable: exit 0.
   - Dangerous: exit 2 with a message that names the fix.
   - Unreadable (a loop, a variable, `eval`): exit 2 and name the plain form the guard can read.
5. Register it with a `timeout`, and use `$CLAUDE_PROJECT_DIR` in the path.
6. Add test cases: the command that caused the incident (refused), the allowed form (allowed), and one near miss of each.
7. Keep a one-line context rule for the reason, so the agent plans around the guard.

## Say how to fix it

A refusal goes back to the agent as text, and the agent's next call depends on what that text says.

| Refusal message | Likely next call |
|---|---|
| `Blocked.` | The same command again, or a variation that dodges the guard |
| `Pattern kills are not allowed.` | A different pattern kill, such as `killall` |
| `Pattern kills are refused on shared machines. Find the PID with pgrep -a <name> and run kill <pid>, or kill your own process group with kill -- -<pgid>.` | `pgrep -a pytest`, then `kill 4121` |

### What a good message contains

1. What was refused, in the agent's terms.
2. Why, in one clause, so the agent does not try the same thing another way.
3. The exact allowed form, ready to run.

#### Example: a guard with three fixes in its message

```bash
#!/usr/bin/env bash
# PreToolUse guard: refuse a pipe that hides a failed command's exit status
cmd=$(jq -r '.tool_input.command')
if [[ "$cmd" =~ (git\ push|git\ commit)[^|]*\|\ *(head|tail|grep) ]]; then
  cat >&2 <<'MSG'
A pipe reports only the last command's status, so a failed push would read as success.
Fix one of three ways:
  1. Run the command without the pipe.
  2. Add `set -o pipefail;` before it.
  3. Redirect output to a file and read the file in a second call.
MSG
  exit 2
fi
```

## Refuse what you cannot prove safe

A guard that parses shell commands meets commands it cannot fully read: loops, variables, command substitution, `eval`. It has two choices.

| Policy | Safe commands | Dangerous commands | Agent experience |
|---|---|---|---|
| Allow what it cannot read | All pass | Some pass hidden in a loop or variable | No extra retries; some dangerous commands run |
| Refuse what it cannot read | Some refused | None pass | Extra retries, each with a fix in the message |

Refusing is the safe default. The cost is false refusals, so the message must say what the guard could not read and name the plain form it can.

#### Example: a false refusal handled well

```text
● Bash(for d in */; do git -C "$d" status; done)
  ⎿  PreToolUse hook refused: this command runs git with a directory computed at
     runtime, so the guard cannot show it stays inside your checkout. Run git
     on one named path per call.

● Bash(git -C ./api status)
  ⎿  On branch main, nothing to commit
```

The isolation guard in my harness refuses loops and commands built from variables for this reason. Each false refusal costs one retry.

## Gate the step on the check

A check protects you only if the risky step cannot run after the check fails.

```bash
# Wrong: the add runs whatever the check said
python check_conflicts.py merged.jsonl
git add merged.jsonl

# Right: the add is unreachable when the check fails
python check_conflicts.py merged.jsonl && git add merged.jsonl
```

Conflict markers reached a repository with every exit code green this way: the check refused correctly, and the next line staged the file anyway.

> Read every script that calls a check and ask: if the check fails here, can the next line still run? Write `check && step`, or put `set -euo pipefail` at the top of the script.

`set -e` has gaps of its own:

| Case | Does `set -e` stop the script |
|---|---|
| `check` fails on its own line | Yes |
| `check \| tee log` fails, without `pipefail` | No: the pipeline's status is `tee`'s |
| `if check; then ...` | No: a failure in a condition is expected |
| `check \|\| echo failed` | No: the `\|\|` handles the failure |

The `check && step` form does not depend on any of these.

### Pipes hide failures

A pipeline reports the exit status of its last command:

```text
$ git push origin main | tail -3
error: failed to push some refs
$ echo $?
0
```

- `tail` succeeded, so the pipeline reports 0 even though the push failed.
- Behind a pipe, a failed push and a failed issue filing were both reported to me as successes.
- With `set -o pipefail` the pipeline fails if any command in it fails; the pipe-status guard above refuses the pipe where the result matters.

## Use the tool that already exists

Before writing a new check, look for one that already enforces the rule:

1. Search the repository with the names the code uses, such as the command, the file pattern or the error text.
2. Search the hook registrations and CI jobs for the same rule.
3. If a check exists, extend it with the new case and a test for it.

A new script that repeats an existing check loses the cases the old one learned to handle, and two checks for one rule drift apart.

## Guards from broken rules

Each of these started as a written rule in a context file or a brief; an agent broke it anyway, and the fix went into the system as a guard. Each entry lists the rule, the incident, what the guard refuses and the message the agent sees.

### The process-kill guard

- Written rule: never kill processes by name pattern on a shared machine. It was in the agent's brief.
- Incident: an agent ran `pkill -f` and stopped three test runs that belonged to other sessions.
- Guard: refuses `pkill`, `killall` and similar pattern kills before they run.
- Refusal the agent sees:

```text
Pattern kills are refused on shared machines. Find the PID with pgrep -a <name>
and run kill <pid>, or kill your own process group with kill -- -<pgid>.
```

### The shell-tracing guard

- Written rule: never print secrets.
- Incident: an agent turned on shell tracing to debug a script. Tracing prints every command with its expanded values, and the session output showed API keys.
- Guard: refuses trace flags such as `set -x` and `bash -x` in commands.
- Refusal the agent sees:

```text
Shell tracing prints expanded values, including secrets. Check a variable by
name instead: [[ -n "${API_KEY:-}" ]] && echo set
```

### The wildcard-delete guard

- Written rule: keep scratch files in a private directory.
- Incident: a sub-agent cleaned up with a wildcard under a shared temporary directory, and the pattern matched files that belonged to the agent that launched it.
- Guard: refuses wildcard deletes under shared scratch locations.
- Refusal the agent sees:

```text
Wildcard deletes under a shared temporary directory are refused. Create a
private directory with d=$(mktemp -d), work inside it, and remove only "$d".
```

### The pipe-status guard

- Written rule: check the exit code of every command whose success matters.
- Incident: a failed push and a failed issue filing were both reported as successes, because each command's output went through a pipe.
- Guard: refuses a pipe after commands such as `git push` and `git commit`.
- Refusal the agent sees: the three fixes listed [earlier on this page](#example-a-guard-with-three-fixes-in-its-message).

### The evidence-citation guard

- Written rule: cite only evidence that is tracked in git.
- Incident: committed documents cited evidence in ignored files; a later cleanup deleted the files, and the documents pointed at nothing.
- Guard: refuses an edit to a committed document when a cited evidence path is not tracked in git.
- Refusal the agent sees:

```text
This edit cites an untracked path as evidence. Commit the evidence file, or cite
a tracked file that shows the same result.
```

### The pattern

| Guard | Written rule before | Incident | Code now |
|---|---|---|---|
| Process kill | "Never kill by pattern" | Three test runs stopped | Refuses; names one-PID form |
| Shell tracing | "Never print secrets" | Keys in session output | Refuses trace flags |
| Wildcard delete | "Keep scratch private" | Launcher's files deleted | Refuses; names `mktemp -d` |
| Pipe status | "Check exit codes" | Failures read as success | Refuses; lists three fixes |
| Evidence citation | "Cite tracked files only" | Documents cited deleted files | Refuses the edit |

In each row the rule was written down and was in the agent's context when it was broken. The written rule stays next to the guard and explains the reason for it.

## Testing a guard

```text
# tests/guards/kill-guard.cases: one command per line, the expected result first
refuse  pkill -f pytest
refuse  killall node
allow   kill 4121
allow   kill -- -4100
allow   pgrep -a pytest
```

A small runner feeds each command to the guard as event JSON and checks the exit code:

```python
import json, pathlib, subprocess

GUARD = ".claude/guards/no-pattern-kill.sh"
CASES = pathlib.Path("tests/guards/kill-guard.cases")

def test_kill_guard_cases():
    for line in CASES.read_text().splitlines():
        if not line.strip() or line.startswith("#"):
            continue
        expected, command = line.split(maxsplit=1)
        event = json.dumps({"tool_name": "Bash", "tool_input": {"command": command}})
        rc = subprocess.run([GUARD], input=event, text=True, capture_output=True).returncode
        assert rc == (2 if expected == "refuse" else 0), f"{expected}: {command} (exit {rc})"
```

- Run it in the commit hook or CI, so a change that weakens the guard fails before it lands.
- Add a case each time the guard misses a command form or refuses a safe one.

## Pitfalls

- A refusal that says only "blocked", which gets a retry or a workaround.
- Allowing commands the guard cannot parse, which lets a dangerous command through inside a loop or a variable.
- A check and the risky step on separate lines of a script.
- A guard with no test cases, which stops refusing when a new command form appears and nobody notices.
- Writing a second guard for a rule another check already enforces.

## Sources

- Hook exit codes and feedback: [Claude Code hooks reference](https://code.claude.com/docs/en/hooks.md), verified 2026-11-09 (partial read of a long page).
- The five guards, their incidents and the isolation guard are from the author's own harness, described conceptually. The scripts, transcripts and test cases are illustrations written for this page. The sequence diagram was made with archify for this page.

{% include page_nav.html %}
