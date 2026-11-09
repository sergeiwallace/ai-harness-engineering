---
title: "Background work and headless runs"
parent: "Claude Code and Codex basics"
nav_order: 4
---

{% include page_nav.html %}

# Background work and headless runs

As of 2026-11-09. Part of [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}). Slide: [stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

Two features change how an agent spends time it is not thinking. Background commands and the Monitor tool let an interactive session start something slow and react the moment it says something useful. Headless runs take the interactive session away altogether, so the client can run inside a script or a CI job.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Background commands

### What changes

A shell command can run with `run_in_background: true`. The agent gets a handle back at once and keeps working. Without that, a forty-minute test run either blocks the session or gets checked every few turns, and each check spends context on "still running".

![Sequence: the agent starts the suite in a background shell and gets a handle at once, asks Monitor to watch the log, reads code meanwhile, receives the FAILED line the moment it is printed, opens the failing test, then receives EXIT 1 and reads the full log only at the end]({{ '/assets/images/03-claude-code-and-codex/monitor-sequence.svg' | relative_url }})

*Waiting on a test run. The agent works while the suite runs and a failure reaches it the moment it is printed. Every streamed line enters the context, so the monitor is filtered to the lines that change the next step. Diagram made with archify.*

## Monitor

### What it does

The Monitor tool watches a background command's output or a file path and feeds each new line back to the agent as it arrives. The agent does not poll. Monitors stop when the session ends.

### Waiting on a test run

```text
● Bash(python -m pytest tests -q > /tmp/run.log 2>&1; echo "EXIT $?" >> /tmp/run.log)
  run_in_background: true

● Monitor(command: tail -f /tmp/run.log | grep --line-buffered -E "FAILED|EXIT")
  ⎿  FAILED tests/test_sync.py::test_lock_release
  ⎿  EXIT 1
```

The agent writes the full output to a file and monitors only the failure and exit lines. It sees the failure as soon as it is printed.

### Patterns

| Pattern | Monitor command | Good for |
|---|---|---|
| Filtered log | `tail -f run.log \| grep --line-buffered -E "ERROR\|FAILED\|EXIT"` | Builds and test runs |
| Status file | `tail -f status.txt` where another process writes one line per state change | Deploys, long jobs |
| Directory watch | a loop that prints a line when an expected file appears | Waiting on another process's output |

### Trade-offs

#### Context cost

Every line a monitor feeds back goes into the context. A monitor on a chatty log fills the window with noise. Point it at filtered output, and keep the full output in a file the agent reads only when needed.

#### Buffering

`grep` buffers its output when it writes to a pipe, so lines can arrive late or all at once. `--line-buffered` makes each match arrive as it happens.

#### Lifetime

A monitor ends with the session. For something that must be watched after you close the terminal, use `/schedule` or an external job.

> Use a monitor for something the session started. For something outside the session, such as a pull request's checks, `/loop` is the better fit. See [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}#loop).

## Headless runs

### claude -p

`claude -p "<prompt>"` runs one task without the interactive terminal and prints the result. Skills a person invokes work in this mode; terminal-only built-in commands such as `/login` do not. A broken settings file shows no dialog in this mode; Claude Code skips it and continues, so run `claude doctor` afterwards if a setting seems ignored.

### --bare

`--bare` skips hooks, skills, plugins and `CLAUDE.md`. A job then runs the same way on every machine, whatever is installed there.

| | `claude -p` | `claude -p --bare` |
|---|---|---|
| Context files | Loaded | Skipped |
| Hooks | Run | Skipped |
| Skills and plugins | Available | Skipped |
| Use for | Scripted work that should follow the project's rules | Evaluations and jobs that must not depend on the machine |

### A CI job

```yaml
- name: Summarise the failing tests
  run: claude -p --bare "Read test-results.xml and list each failing test with a one-line cause" > summary.txt
```

### A project-aware script

```bash
# Uses the project's CLAUDE.md and hooks, so the agent follows team rules
claude -p "Update CHANGELOG.md from the commits since the last tag. Do not edit any other file."
```

<details markdown="1">
<summary>Choosing between the two for an evaluation</summary>

An evaluation compares runs. If the runs load whatever context files and hooks happen to be on the machine, a difference in the result can come from the setup and not from the change you are testing. `--bare` removes that variable. When the thing under test is the harness itself, run without `--bare` and pin the harness version instead. See [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}).

</details>

## Pitfalls and lessons

- An unfiltered monitor fills the context with log noise.
- A headless run hides settings errors; check `claude doctor`.
- `--bare` also drops the safety rules your hooks enforce. Do not use it for jobs that edit or push.

## Sources

- Monitor tool and background commands: <https://code.claude.com/docs/en/tools-reference>, verified 2026-11-02.
- Headless mode and `--bare`: <https://code.claude.com/docs/en/headless.md>, verified 2026-11-09.
- Settings errors in `-p` runs: <https://code.claude.com/docs/en/settings.md>, read 2026-11-09.
- Commands and scripts are illustrations written for this page.

{% include page_nav.html %}
