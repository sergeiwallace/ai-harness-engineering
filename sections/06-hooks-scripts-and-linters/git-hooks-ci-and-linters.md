---
title: "Git hooks, CI and linters"
parent: "Hooks, scripts and linters"
nav_order: 3
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Git hooks, CI and linters

[Tool-call hooks]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}) check what an agent is about to do. Git hooks, CI gates and linters check what it produced, for agents and people alike.

| Layer on [the enforcement ladder]({{ '/sections/06-hooks-scripts-and-linters/enforcement-ladder.html' | relative_url }}) | Checks | Proactive use | Reactive use |
|---|---|---|---|
| 3 | Linters and git hooks, at commit and push | Formatting, naming, import boundaries, commit message format | A secret, a debug statement, a known bad pattern |
| 4 | Tests and CI, before the change lands | The behaviour the task asked for, required sections in docs | A regression that happened before |

This page covers each check, how to split work between them, ratchets for adopting strict rules in an existing codebase, and what gates cost. It is part of [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

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

Put the fastest checks earliest.

- A pre-commit hook runs on every commit, and agents commit often, so a two-minute hook costs two minutes many times a day.
- Tests for the files a change touches fit in pre-push.
- The full suite, builds and scans belong in CI.

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

Git runs a script at fixed points, and a non-zero exit stops the operation:

| Git hook | Runs | Typical checks |
|---|---|---|
| `pre-commit` | Before the commit is made | Format, lint, secret scan, fast tests |
| `commit-msg` | After the message is written | Message format, required trailers |
| `pre-push` | Before the push | Tests for the changed files |

Two ways to share them with a team:

- Keep the scripts in the repository and point git at them with `git config core.hooksPath .githooks`.
- Use a framework such as `pre-commit`, which installs the hooks from a config file:

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

A commit-message hook can require structured trailers.

- In my harness one requires each commit to record which model and agent wrote it.
- A later review can then tell agent work from human work, and one model's work from another's.
- The hook refuses a commit without the trailers and prints the lines to add:

```text
fix: retry the upload when the token expires

Model: <model name>
Agent: <agent role>
```

## CI gates

A CI gate runs on a server after a push or before a merge. It is the slowest check and the only one that runs on a clean machine, so it catches what depends on a local setup:

- A file that exists only on one laptop.
- A dependency installed by hand and missing from the lock file.
- A test that passes only with a warm cache or a running local service.

### What belongs in CI

| Belongs in CI | Why there |
|---|---|
| The full test suite | Too slow for every commit |
| Builds for every target | Needs a clean environment |
| Dependency and licence scans | Needs network and a database |
| Checks of the result against the history | "Release notes list every merged change" |

<details markdown="1">
<summary>When CI does not gate the merge</summary>

Some teams run CI but merge on a green local gate, for speed or because CI is not trusted yet.

- The local gate is then the real gate, and it must be as strict as CI would be.
- Say which one gates in the repository's context file, so agents treat the right result as final.

</details>

## Linters

A linter checks code for style and common mistakes without running it. For agents it works as a fast reviewer:

- It runs in under a second on a changed file.
- It gives the same result every time for the same code.
- Its messages name the rule, the file and the line, so the fix is clear.

### Linters agents learn from

```text
src/upload.py:42:5: B904 Within an except clause, raise exceptions with `raise ... from err`
src/upload.py:57:1: C901 `retry_upload` is too complex (14 > 10)
```

1. The agent reads the rule code and the line.
2. It fixes the line and runs the linter again.
3. The commit goes through when the linter is clean.

A lint rule that agents break often catches each break in seconds, at the commit where it happened.

### Custom lint rules

A team rule that a program can check can be a custom lint rule, which then runs at commit time in every session for every agent. One example, with an import check:

```toml
# pyproject.toml: no direct database imports outside the repository layer
[tool.importlinter]
root_package = "app"
include_external_packages = true

[[tool.importlinter.contracts]]
name = "Only the repository layer talks to the database"
type = "forbidden"
source_modules = ["app.api", "app.services"]
forbidden_modules = ["sqlalchemy"]
```

The same rule as a context-file line would be followed unevenly; as a lint rule it fails the commit with the module that broke it.

## Tests as the last layer

Tests are layer 4: they check what the change does, after every earlier layer has had its turn.

- Fast unit tests fit in pre-commit or pre-push; the full suite runs in CI.
- A test written from the task's acceptance criteria checks the behaviour the agent was asked for.
- A test for each guard checks that the earlier layers still refuse what they should: the [guard test runner]({{ '/sections/06-hooks-scripts-and-linters/writing-a-guard.html' | relative_url }}#testing-a-guard) runs in CI like any other test.

Section 12 covers [testing agent-written code]({{ '/sections/12-testing/' | relative_url }}) in depth.

## Ratchets

A ratchet is a check that lets a quality number only improve.

- Typical numbers: lint warnings, untyped functions, files over a size limit, skipped tests.
- Each number may go down but never up.
- It lets a team adopt a strict rule in a large codebase without fixing everything first.

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

# count_lint, count_untyped and count_long are your own counting functions
baseline = json.load(open("quality-baseline.json"))
current = {"lint_warnings": count_lint(), "untyped": count_untyped(), "long_files": count_long()}
failed = [k for k, v in current.items() if v > baseline[k]]
if failed:
    sys.exit(f"Ratchet failed: {', '.join(f'{k} {current[k]} > {baseline[k]}' for k in failed)}")
json.dump({k: min(v, baseline[k]) for k, v in current.items()}, open("quality-baseline.json", "w"))
```

## What gates cost

| Cost | What happens | Mitigation |
|---|---|---|
| Time | Slow hooks multiply across every agent commit | Fast checks first; slow ones in CI |
| Flakiness | A check that fails at random leads people and agents to retry without reading the failure | Fix or quarantine flaky tests |
| Bypass | A gate that cannot be passed leads people and agents to skip it | Every refusal says what will pass |
| Drift | Local hooks and CI check different things | One config shared by both |

## Pitfalls

- A pre-commit hook slow enough that agents look for a way around it.
- A harness that lets an agent pass `--no-verify`.
- CI that checks something the local gate does not, so agents learn of failures an hour late.

## Sources

- Git hooks and `core.hooksPath`: [Git documentation for githooks](https://git-scm.com/docs/githooks) (general reference, not re-read for this page).
- The pre-commit framework: [pre-commit framework site](https://pre-commit.com/) (general reference, not re-read for this page).
- The commit trailer check and the guard on skipped hooks are from the author's own harness, described conceptually. The configs, transcripts and ratchet code are illustrations written for this page.

{% include page_nav.html %}
