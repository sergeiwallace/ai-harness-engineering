---
title: "Writing a guard"
parent: "Hooks"
grand_parent: "Hooks, scripts and linters"
nav_order: 1
---

{% include page_nav.html %}

# Writing a guard

As of 2026-11-09.

A guard is a [hook]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}) that refuses an action. Whether agents can work with a guard depends on three things: what its refusal says, what it does with commands it cannot read, and whether the risky step is unreachable when the check fails. This page covers each, then tells the stories of five guards in my harness that started as written rules agents broke. It is a sub-topic of [Hooks]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## A refusal and a recovery

![Sequence diagram with four lanes: Model, Client, PreToolUse hook and Shell. The model asks for pkill -f pytest; the hook exits 2 with the message that pattern kills are refused; the client returns the block to the model. The model runs pgrep -a pytest, which the hook allows, and gets process 4121. The model runs kill 4121; the hook allows one PID; only that process stops.]({{ '/assets/images/06-hooks-scripts-and-linters/guard-refusal-sequence.png' | relative_url }})
*The refusal message in the red arrow names the allowed form, and the agent's next two calls use it.*

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
| Allow what it cannot read | All pass | Some pass hidden in a loop or variable | Smooth until the incident |
| Refuse what it cannot read | Some refused | None pass | Extra retries, each with a fix |

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

> Read every script that calls a check and ask: if the check fails here, can the next line still run? Write `check && step`, or `set -e` at the top of the script.

### Pipes hide failures

`git push origin main | tail -3` reports the exit status of `tail`, which is 0 even when the push failed. Behind a pipe, a failed push and a failed issue filing were both reported to me as successes. That is the reason for the pipe-status guard above.

## Use the tool that already exists

Before writing a new check, search the repository for a tool that already enforces the rule, using the names the code uses. A new script that repeats an existing check loses the cases the old one learned to handle, and two checks for one rule drift apart.

## Guards from broken rules

Each of these started as a written rule in a context file or a brief. An agent broke it anyway, and the rule became code.

### The process-kill guard

#### What happened

The rule against killing processes by name pattern was in the agent's brief. An agent ran `pkill -f` on a shared machine and stopped three test runs that belonged to other sessions.

#### What the guard does

It refuses `pkill`, `killall` and similar pattern kills before they run, and tells the agent to find one process id and kill it, or kill its own process group.

### The shell-tracing guard

#### What happened

An agent turned on shell tracing to debug a script. Tracing prints every command with its expanded values, and the session output showed API keys.

#### What the guard does

It refuses trace flags in commands and says to check a variable by name, such as testing whether it is set, without printing its value.

### The wildcard-delete guard

#### What happened

A sub-agent cleaned up its scratch files with a wildcard under a shared temporary directory. The pattern matched files that belonged to the agent that launched it.

#### What the guard does

It refuses wildcard deletes under shared scratch locations and says to make a private directory with `mktemp -d` and remove only that directory.

### The pipe-status guard

#### What happened

A failed push and a failed issue filing were both reported as successes, because each command's output went through a pipe.

#### What the guard does

It refuses the pipe on commands whose success matters and lists the three fixes shown earlier on this page.

### The evidence-citation guard

#### What happened

Committed documents cited evidence in ignored files. A later cleanup deleted the files, and the documents pointed at nothing.

#### What the guard does

It refuses an edit to a committed document when a cited evidence path is not tracked in git, and says to commit the evidence or cite something that is.

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

```bash
# guard test cases: one command per line, the expected result first
refuse  pkill -f pytest
refuse  killall node
allow   kill 4121
allow   kill -- -4100
allow   pgrep -a pytest
```

A small runner feeds each command to the guard as event JSON and checks the exit code. A guard without tests drifts: the harness changes, a new command form appears, and nobody notices the guard stopped refusing it.

## Sources

- Hook exit codes and feedback: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-09 (partial read of a long page).
- The five guards, their incidents and the isolation guard are from the author's own harness, described conceptually. The scripts, transcripts and test cases are illustrations written for this page. The sequence diagram was made with archify for this page.

{% include page_nav.html %}
