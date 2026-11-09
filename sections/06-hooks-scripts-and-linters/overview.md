---
title: "Hooks, scripts and linters"
nav_order: 16
has_children: true
has_toc: false
permalink: /sections/06-hooks-scripts-and-linters/
---

{% include page_nav.html %}

# Hooks, scripts and linters

As of 2026-11-09.

Stop 5 on the talk path, about 4 minutes.

A context-file instruction guides an agent, and the agent applies it unevenly: less as its context fills, when it juggles several tasks, after a task switch, or after a compaction drops the rule. The working principle of this site is a [systems approach to harness engineering]({{ '/sections/02-ai-harnesses/' | relative_url }}#a-systems-approach-to-harness-engineering): a fix goes into the system and is enforced by code wherever a program can check it. A harness enforces its rules in four layers:

1. Instructions in context files, skills and prompts guide the model. They are the second line of defence and never enough on their own.
2. Workflows and tools build the steps into what the agent runs.
3. Hooks, scripts and linters refuse or flag an action at the moment the agent tries it.
4. Validation and tests catch what got past the earlier layers before a change lands.

Layers 2 to 4 do the enforcing. This section covers layers 3 and 4, and the scripts that layer 2 relies on.

![A five-step staircase from least to most enforced: context rule, path rule, skill, hook or guard, test or CI gate. Each step up is more consistent and more costly. A rule goes to the highest step that can check it.]({{ '/assets/images/06-hooks-scripts-and-linters/enforcement-ladder.svg' | relative_url }})

This overview covers each part with a worked example. Three topic pages go deeper:

- [The enforcement ladder]({{ '/sections/06-hooks-scripts-and-linters/enforcement-ladder.html' | relative_url }}): the four layers and five mechanisms in depth, how to place a rule, and what to do when an agent breaks one.
- [Hooks]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}): events, handler types, input and output, exit codes, registration and debugging. Its sub-page [Writing a guard]({{ '/sections/06-hooks-scripts-and-linters/writing-a-guard.html' | relative_url }}) covers refusal messages, parsing limits and the guards in my own harness.
- [Git hooks, CI and linters]({{ '/sections/06-hooks-scripts-and-linters/git-hooks-ci-and-linters.html' | relative_url }}): checks at commit, push and merge, linters, ratchets and the costs of gates.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Why enforce in code

### Instructions drift

A rule in `CLAUDE.md` or `AGENTS.md` is read by the model. Whether it is followed depends on the state of the session:

- Several tasks at once: the model attends to the task in front of it and drops rules about the others.
- A task switch: a rule that applied to the first task is not recalled when the agent comes back to it.
- A nearly full context window: early instructions carry less weight, and a compaction can summarise a rule away.
- A sub-agent: it sees only what its brief carried.

Claude Code's own docs call `CLAUDE.md` "context, not enforced configuration".

### Code checks run every time

A hook or check runs at a fixed point with the same result for the same input. It does not depend on what the model remembers.

| | Written rule | Code check |
|---|---|---|
| Who applies it | The model, when it remembers | The client or git, every time |
| After compaction | May be gone | Unchanged |
| In a sub-agent | Only if the brief carried it | Unchanged |
| Cost | Tokens every turn | Time per check, maintenance |
| Fits | Judgment, preferences, why | Anything a program can decide |

> When Claude skips a rule that must hold every time, the Claude Code skills guide says to move it into a hook.

## The enforcement ladder

The ladder ranks the places a rule can live, from read-by-the-model to checked-by-code.

| Step | Mechanism | Layer | Checked | Example rule |
|---|---|---|---|---|
| 1 | Context rule | 1 Instructions | Never; the model reads it | "Prefer small functions" |
| 2 | Path rule | 1 Instructions | Never; loads for matching files | "In `migrations/`, never edit a merged file" |
| 3 | Skill | 2 Workflows and tools | When it runs | The release procedure |
| 4 | Hook or guard | 3 Hooks, scripts and linters | At the moment the agent acts | "Never force push" |
| 5 | Test or CI gate | 4 Validation and tests | On the result | "Release notes list every merged change" |

Put each rule at the highest step that can check it, and keep a one-line context rule for the reason.

> Adding another detailed rule to a context file each time an agent does something wrong is an anti-pattern. The new rule is followed as unevenly as the old one. The correction goes into a workflow, tool, hook, script, linter or test, and the context line is at most a second layer.

### Example: a broken rule fixed in the system

![Sequence diagram with five lanes: Harness author, Context file, Agent, PreToolUse hook and CI gate. Week 1: the author writes "use the test lock on the shared machine" into the context file; it loads at session start; a compaction drops the rule; the agent runs the suite without the lock and two runs collide. Week 3: the author adds a hook that refuses the suite command without the lock. The agent runs npm run e2e; the hook exits 2 with the wrapped form; the agent runs test-lock npm run e2e; the hook exits 0. After: the context file still says why; the agent pushes; CI runs the full suite on a clean machine and passes.]({{ '/assets/images/06-hooks-scripts-and-linters/rule-climbs-ladder-sequence.png' | relative_url }})
*One rule over three weeks. The written rule was in the window and still broke; the hook's refusal names the wrapped command, so the agent's next call is right.*

The [ladder page]({{ '/sections/06-hooks-scripts-and-linters/enforcement-ladder.html' | relative_url }}) has a placement guide and more examples.

## Hooks

A hook is a handler the client runs at a lifecycle event. Claude Code's hooks reference lists 33 events; Codex has 14.

![A session timeline: SessionStart, UserPromptSubmit, a repeating loop of PreToolUse, the tool call and PostToolUse, then PreCompact and Stop. UserPromptSubmit, PreToolUse and Stop can block.]({{ '/assets/images/06-hooks-scripts-and-linters/hook-lifecycle.svg' | relative_url }})

### The events a harness uses most

| Event | Fires | Typical use |
|---|---|---|
| `SessionStart` | At start or resume | Load state; print reminders |
| `UserPromptSubmit` | When a prompt is sent | Add context; refuse a prompt |
| `PreToolUse` | Before a tool call | Refuse dangerous commands |
| `PostToolUse` | After a tool call | Format edited files; record changes |
| `PreCompact` | Before compaction | Save state the summary might lose |
| `Stop` | When the agent tries to finish | Refuse to stop until a check passes |

### Exit codes

In Claude Code a `PreToolUse` hook and the agent work together like this:

1. The agent asks for a tool call.
2. The client passes the call to the hook as JSON.
3. The hook exits.
   - Exit code 0: the call proceeds.
   - Exit code 2: the call is blocked, and the hook's standard error goes back to the agent.
4. The agent reads the message and makes its next call.

A hook after a tool call cannot undo the call, because the tool already ran. The [process-kill case study](#case-study-the-process-kill-guard) below shows the loop as a diagram.

### Example: a hook that blocks force pushes

#### The script

```bash
#!/usr/bin/env bash
cmd=$(jq -r '.tool_input.command')
if [[ "$cmd" =~ git\ push.*(--force|-f\ ) ]]; then
  echo "Force push refused. Push to a new branch and open a pull request." >&2
  exit 2
fi
```

#### The registration

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash",
        "hooks": [{ "type": "command", "command": ".claude/guards/no-force-push.sh", "timeout": 5 }] }
    ]
  }
}
```

#### What the agent sees

```text
● Bash(git push --force origin feature/login)
  ⎿  PreToolUse hook refused: Force push refused. Push to a new branch and open a pull request.

● Bash(git push origin feature/login-v2)
  ⎿  To github.com:example/app.git
      * [new branch]      feature/login-v2 -> feature/login-v2
```

`/hooks` shows every configured hook and where it comes from. The [hooks page]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}) covers handler types, the JSON a hook receives and returns, matchers and debugging.

## Writing a guard

A guard is a hook that refuses an action. Three habits decide whether agents can work with it.

| Habit | Weak | Strong |
|---|---|---|
| Say how to fix it | `Blocked.` | `Pattern kills are refused. Find the PID with pgrep -a <name> and run kill <pid>.` |
| Refuse what you cannot prove safe | Allow anything the parser does not understand | Refuse it, and name the plain form the guard can read |
| Gate the step on the check | Check on one line, act on the next | `check && act` |

### Example: the check that protected nothing

```bash
# Wrong: the add runs whatever the check said
python check_conflicts.py merged.jsonl
git add merged.jsonl

# Right: the add is unreachable when the check fails
python check_conflicts.py merged.jsonl && git add merged.jsonl
```

Conflict markers reached a repository with every exit code green this way: the check refused correctly, and the next line staged the file anyway.

### Case study: the process-kill guard

![Sequence diagram with four lanes: Model, Client, PreToolUse hook and Shell. First attempt: the model asks for pkill -f pytest; the client sends the event JSON to the hook; the hook exits 2 with the message that pattern kills are refused; the client returns the block and message to the model. The agent follows the fix: it runs pgrep -a pytest, the hook allows it, and the shell returns process 4121. Second attempt: the model asks for kill 4121; the hook allows one PID; the shell stops only that process.]({{ '/assets/images/06-hooks-scripts-and-linters/guard-refusal-sequence.png' | relative_url }})
*The process-kill guard during a real task. The refusal carries the fix, so the agent's next two calls are the right ones.*

The rule was already in the agent's brief. Writing it more firmly would not have helped. The fix was a hook that refuses any pattern kill and tells the agent to kill one process id or one process group. The [guard page]({{ '/sections/06-hooks-scripts-and-linters/writing-a-guard.html' | relative_url }}) has this case in full and four more from my harness.

## Git hooks, CI and linters

Tool-call hooks check actions. Git hooks, CI and linters check the result.

![Sequence diagram with five lanes: Agent, PreToolUse hook, pre-commit, pre-push and CI. Commit: the agent runs git commit; pre-commit reports lint rule B904 at upload.py line 42 and exits 1; the agent tries git commit --no-verify; the PreToolUse hook exits 2 and says to fix what the hooks report; the agent fixes the raise and commits; pre-commit passes in 1.3 seconds. Push: pre-push runs 14 of 1,204 tests near the change and passes in 38 seconds. Merge: CI runs the full suite, build and scan on a clean machine and passes in 11 minutes.]({{ '/assets/images/06-hooks-scripts-and-linters/commit-to-merge-sequence.png' | relative_url }})
*One agent change on its way to merge. The tool-call hook closes the skip, so the git hook holds; the fast checks answer in seconds and the full answer comes before merge.*

| Check | Runs | Catches | Speed |
|---|---|---|---|
| Pre-commit hook | Each commit | Format, lint, secrets, fast tests | Seconds |
| Pre-push hook | Each push | Tests that cover the changed files | Minutes |
| CI gate | After push, before merge | The full suite on a clean machine | Minutes to an hour |
| Ratchet | Any of the above | A quality number getting worse | Seconds |

### Example: a ratchet

A ratchet lets a number only improve, so a strict rule can be adopted in a large codebase without fixing everything first.

```text
$ make quality
lint warnings: 412 (baseline 415)  OK, baseline lowered to 412
untyped functions: 88 (baseline 88) OK
files over 800 lines: 13 (baseline 12)  FAIL: one more than the baseline
```

An agent cannot skip git hooks unless it passes `--no-verify`, and a tool-call hook can refuse that flag. The [git hooks, CI and linters page]({{ '/sections/06-hooks-scripts-and-linters/git-hooks-ci-and-linters.html' | relative_url }}) covers each check in depth.

## Scripts

A script turns a procedure that would be prose into code: the same steps, the same order, an exit code. A skill whose step says "run `release_check.py`" gets the same answer every run. When an agent keeps getting a multi-step procedure wrong, the fix is often a script that does the procedure, and a context line or skill step that says to run it.

### Example: a procedure into a script

| Before: a context paragraph | After: a script plus one line |
|---|---|
| "Before a release, check the version in three files matches, the changelog has an entry for it, and the tag does not exist yet." | `release_check.py` checks all three and exits 1 with the mismatch. The context line: "Run `release_check.py` before tagging." |

## What guards cost

| Cost | What happens | Mitigation |
|---|---|---|
| False refusals | A safe command the guard cannot prove safe is refused | Say so in the message; offer the plain form |
| Time | Every hook adds time to every call it checks | Narrow matchers; fast scripts |
| Hangs | A hung hook blocks all work | A timeout on every hook; visible failure |
| Tokens | Injected text is read on every event | Inject only when it applies |
| Maintenance | Guards go stale as the harness changes | Tests for the guards; one registration manifest |

> A guard that refuses should also say what will pass, so the next attempt fixes the command.

## Examples from my own harness

- Guards that started as written rules agents broke: a process-kill guard, a shell-tracing guard, a wildcard-delete guard, a pipe-status guard and an evidence-citation guard. The [guard page]({{ '/sections/06-hooks-scripts-and-linters/writing-a-guard.html' | relative_url }}#guards-from-broken-rules) tells each story.
- A worktree guard that stops a sub-agent from writing outside its own checkout, including running git against another directory.
- Commit and push checks that run the formatter, the linter and the tests the diff touches, and require each commit to record which model and agent wrote it.
- Hooks registered from one manifest, with a test that every hook file is registered and every registration names a file.

To start: turn the rule an agent has broken twice into a hook.

## Pitfalls and lessons

- A refusal without a fix gets a retry or a workaround.
- A check and a risky step on separate lines protect nothing.
- A hook without a timeout can stop all work.
- Several guards in my harness started as written rules that agents broke. Each break was fixed in the system with a guard, and the written rule stayed as the reason.

## Sources

- Claude Code hooks, events, exit code 2, handler types: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-09 (partial read of a long page).
- "context, not enforced configuration": <https://code.claude.com/docs/en/memory.md>, verified 2026-11-09.
- Moving a must-hold rule into a hook: <https://code.claude.com/docs/en/skills.md>, verified 2026-11-09.
- Codex hooks: <https://learn.chatgpt.com/docs/hooks.md>, verified 2026-11-08.
- The guards, the manifest and the isolation guard are from the author's own harness, described conceptually. The example scripts, settings and transcripts are illustrations written for this page. The ladder and timeline diagrams were drawn for this page.

{% include page_nav.html %}
