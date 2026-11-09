---
title: "Git hooks, CI and linters"
parent: "Hooks, scripts and linters"
nav_order: 3
---

{% include page_nav.html %}

# Git hooks, CI and linters

As of 2026-11-09.

[Tool-call hooks]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}) check what an agent is about to do. Git hooks, CI gates and linters check what it produced. They apply to agents and people alike, and they are the top step of [the enforcement ladder]({{ '/sections/06-hooks-scripts-and-linters/enforcement-ladder.html' | relative_url }}). This page covers each check, how to split work between them, ratchets for adopting strict rules in an existing codebase, and what gates cost. It is part of [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Where each check runs

| Check | Runs | Machine | Catches | Typical time |
|---|---|---|---|---|
| Pre-commit hook | Each commit | Local | Format, lint, secrets, fast tests | Seconds |
| Commit-message hook | Each commit | Local | Message format, required trailers | Under a second |
| Pre-push hook | Each push | Local | Tests that cover the changed files | Minutes |
| CI gate | After push, before merge | Clean server | The full suite, builds, scans | Minutes to an hour |

### Split the work by speed

Put the fastest checks earliest. A pre-commit hook that takes two minutes is run hundreds of times a day by agents, and its cost multiplies. A check that needs the full suite belongs in pre-push or CI.

#### Worked example: one change through every check

![Sequence diagram with five lanes: Agent, PreToolUse hook, pre-commit, pre-push and CI. The first commit is refused by the linter; the agent tries to skip the hooks and the tool-call hook refuses; the agent fixes the lint finding and the commit passes. The push runs the tests near the change; CI runs the full suite on a clean machine.]({{ '/assets/images/06-hooks-scripts-and-linters/commit-to-merge-sequence.png' | relative_url }})
*The same change as the log below, including the refused first commit the log does not show.*

The log of the second, accepted commit and what followed:

```text
$ git commit -m "fix: retry the upload when the token expires"
pre-commit: ruff format ........ ok (0.4s)
pre-commit: ruff check ......... ok (0.6s)
pre-commit: secret scan ........ ok (0.3s)
commit-msg: conventional title . ok

$ git push origin fix/upload-retry
pre-push: tests for changed files (14 of 1,204) ... ok (38s)

CI #4182: full suite ........ ok (11m)
CI #4182: build ............. ok
CI #4182: dependency scan ... ok
```

The agent got answers in seconds for most problems, a minute for the tests near its change, and the full answer before merge.

## Git hooks

### What they are

Git runs a script at fixed points: before a commit is made, when its message is written, before a push. A non-zero exit stops the commit or push. Teams keep hook scripts in the repository and point git at them with `core.hooksPath`, or use a framework such as `pre-commit` that installs them from a config file.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff-format
      - id: ruff
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.0
    hooks:
      - id: gitleaks
```

### Agents and `--no-verify`

An agent cannot skip git hooks unless it passes `--no-verify`, or sets the hooks path to nothing. A tool-call hook can refuse both, so the git hook holds for agents.

```text
● Bash(git commit --no-verify -m "wip")
  ⎿  PreToolUse hook refused: skipping git hooks is not allowed. Run the commit
     without --no-verify and fix what the hooks report.
```

> A git hook protects you only if nothing in the harness can skip it. Check for `--no-verify`, `-n` on commit, and an emptied `core.hooksPath`.

### Commit checks for agent work

A commit-message hook can require structured trailers. In my harness one requires each commit to record which model and agent wrote it, so a later review can tell agent work from human work and one model's from another's.

```text
fix: retry the upload when the token expires

Model: <model name>
Agent: <agent role>
```

## CI gates

A CI gate runs on a server after a push or before a merge. It is the slowest check and the only one that runs on a clean machine, so it catches what depends on a local setup: a file that exists only on one laptop, a dependency installed by hand, a test that passes only with a warm cache.

### What belongs in CI

| Belongs in CI | Why there |
|---|---|
| The full test suite | Too slow for every commit |
| Builds for every target | Needs a clean environment |
| Dependency and licence scans | Needs network and a database |
| Checks of the result against the history | "Release notes list every merged change" |

<details markdown="1">
<summary>When CI does not gate the merge</summary>

Some teams run CI but merge on a green local gate, for speed or because CI is not trusted yet. Then the local gate is the real gate, and it must be as strict as CI would be. Say which one gates in the repository's context file, so agents do not treat a red CI run as advisory or a green local run as enough when it is not.

</details>

## Linters

A linter checks code for style and common mistakes without running it. For agents a linter is a fast reviewer: it runs in under a second, it is the same every time, and its messages say exactly what to fix.

### Linters agents learn from

```text
src/upload.py:42:5: B904 Within an except clause, raise exceptions with `raise ... from err`
src/upload.py:57:1: C901 `retry_upload` is too complex (14 > 10)
```

An agent reads the rule code and the line, fixes it, and runs the linter again. A rule that agents break often is worth turning on, because each break is caught in seconds.

### Custom lint rules

A team rule that a program can check, such as "no direct database calls outside the repository layer", can be a custom lint rule. It then runs at commit time, in every session, for every agent.

## Ratchets

A ratchet is a check that lets a quality number only improve. The number of lint warnings, untyped functions or files over a size limit may go down but never up. A ratchet lets a team adopt a strict rule in a large codebase without fixing everything first.

### Worked example: a ratchet run

```text
$ make quality
lint warnings: 412 (baseline 415)  OK, baseline lowered to 412
untyped functions: 88 (baseline 88) OK
files over 800 lines: 13 (baseline 12)  FAIL: one more than the baseline
```

### How it works

1. Store the baseline numbers in a committed file.
2. The check counts the current numbers.
3. A number above its baseline fails.
4. A number below its baseline passes and lowers the baseline in the same commit.

#### The check in a few lines

```python
import json, sys

baseline = json.load(open("quality-baseline.json"))
current = {"lint_warnings": count_lint(), "untyped": count_untyped(), "long_files": count_long()}
failed = [k for k, v in current.items() if v > baseline[k]]
if failed:
    sys.exit(f"Ratchet failed: {', '.join(f'{k} {current[k]} > {baseline[k]}' for k in failed)}")
json.dump({k: min(v, baseline[k]) for k, v in current.items()}, open("quality-baseline.json", "w"))
```

> A ratchet suits agents well: an agent that adds a warning is told at once, and an agent that removes one moves the baseline for everyone.

## What gates cost

| Cost | What happens | Mitigation |
|---|---|---|
| Time | Slow hooks multiply across every agent commit | Fast checks first; slow ones in CI |
| Flakiness | A check that fails at random teaches everyone to retry blindly | Fix or quarantine flaky tests |
| Bypass | An unpassable gate teaches people and agents to skip it | Every refusal says what will pass |
| Drift | Local hooks and CI check different things | One config shared by both |

## Pitfalls

- A pre-commit hook slow enough that agents look for a way around it.
- A harness that lets an agent pass `--no-verify`.
- CI that checks something the local gate does not, so agents learn of failures an hour late.

## Sources

- Git hooks and `core.hooksPath`: <https://git-scm.com/docs/githooks> (general reference, not re-read for this page).
- The pre-commit framework: <https://pre-commit.com/> (general reference, not re-read for this page).
- The commit trailer check and the guard on skipped hooks are from the author's own harness, described conceptually. The configs, transcripts and ratchet code are illustrations written for this page.

{% include page_nav.html %}
