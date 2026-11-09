---
title: "Detail"
parent: "Hooks, scripts and linters"
nav_order: 1
---

# Hooks and scripts in detail

As of 2026-11-09.

## Purpose

Explain deterministic controls: code that runs at fixed points whether or not the model remembers the rule.

## Key points

- Enforce as much as possible in code. An agent follows a context-file instruction inconsistently over time: more so as its context fills, when it juggles several tasks, and when it switches away and forgets to come back. Code runs the same way every time.
- The options, from soft to hard enforcement, and when each fits:
  1. Context rule (a line in `CLAUDE.md` or `AGENTS.md`): preferences and judgment calls that apply everywhere and that no program can check. Cheapest, and the least consistently followed.
  2. Path rule: the same, for rules that apply only to some files. It keeps the always-loaded context small, and it loads only when a file tool touches a matching path.
  3. Skill: a procedure with fixed steps, followed consistently when it runs. Nothing forces it to run.
  4. Hook or guard: an action that must always or never happen, checked at the moment the agent tries it. It runs every time and can block the action with a message that says how to fix it.
  5. Test or CI gate: a property of the result, such as working code, clean files or complete commit records, checked at commit, push or merge. It catches what got past the earlier steps, later and more slowly.
- A hook is a handler the client runs at a lifecycle event. Claude Code's hooks reference lists 33 events, such as before a tool call, after a tool call, at session start, before compaction and when the agent stops. Codex has 14.
- A hook can block the action at many of these events. In Claude Code, exit code 2 means "stop, don't do this", and the hook's message goes back to the agent. A hook after a tool call cannot undo it, because the tool already ran.
- A hook can be a shell command, an HTTP call, a tool call, a prompt to a model or an agent.
- A refusal message that says exactly how to fix the problem gets a correct second attempt.
- A check protects you only if the risky step cannot run after the check fails. When the check and the risky step are separate lines in a script, the step still runs after a failed check. Write `check && step`.
- `/hooks`: Shows every configured hook and where it comes from, read-only.

## Examples from my own harness

- Instructions that became guards in my harness, each after agents broke the written rule:
  - A process-kill guard. The rule against killing processes by name pattern was already in the agent's brief when an agent ran `pkill -f` on a shared machine and killed three unrelated test runs.
  - A shell-tracing guard, after shell tracing echoed API keys into the session.
  - An evidence-citation guard. The rule was to cite evidence only from files in git, and committed documents kept pointing at ignored files that a cleanup would delete.
  - A wildcard-delete guard, after a sub-agent deleted many of its launcher's files with a wildcard under a shared temporary directory.
  - A pipe-status guard. A command piped into another reports only the last command's status, so a failed push and a failed issue filing were both reported as successes. The guard refuses that pipe and lists three ways to fix it.
- A worktree guard that stops a sub-agent from writing outside its own checkout.
- Commit and push checks, including one that requires each commit to record which model and agent wrote it.

To start: turn the rule an agent has broken twice into a hook.

## Pitfalls and lessons

- Guards cost something. They can block legitimate work, they need maintenance as the harness changes, and they add time to every call they check. A guard that cannot prove a command safe refuses it, even when it is safe.
- A hook that runs on every launch or tool call can block all work when it hangs. Give each hook a time limit and a visible failure.
- Hooks that inject text add tokens on every call. Scope injected text to the sessions it concerns.
- A script can check a file, refuse, and then commit it anyway on the next line. That is how conflict markers reach a repository with every exit code green.
- Reflexivity example: several guards in my harness started as written rules that agents broke.

## Sources

- Claude Code hooks, events, exit code 2, handler types: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-08 (partial read of a long page).
- Codex hooks: <https://learn.chatgpt.com/docs/hooks.md>, verified 2026-11-08.
