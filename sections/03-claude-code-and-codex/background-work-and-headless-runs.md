---
title: "Background work and headless runs"
parent: "Claude Code and Codex basics"
nav_order: 4
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# Background work and headless runs

Two features change how an agent spends the time it is not thinking:

- Background commands let an interactive session start something slow and carry on; the result wakes the session when it finishes. The Monitor tool adds a way to react to lines before then.
- Headless runs (`claude -p`) remove the interactive session, so the client runs inside a script or a CI job.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Background commands

### What changes

Without a background command, a forty-minute test run blocks the session until it ends. With one:

1. The agent runs the shell command with `run_in_background: true` and gets a handle back at once.
2. It keeps working, or ends its turn and the session goes idle.
3. When the command finishes, its result wakes the session as a new turn, automatically.
   - No goal, loop or polling is needed for this.
   - A sub-agent the session started works the same way: its result arrives when it finishes.
4. A Monitor is optional, for reacting to lines before the command finishes, such as the first failure in a long suite.

![Sequence: the agent starts the suite in a background shell and gets a handle at once, asks Monitor to watch the log, reads code meanwhile, receives the FAILED line the moment it is printed, opens the failing test, then receives EXIT 1 and reads the full log only at the end]({{ '/assets/images/03-claude-code-and-codex/monitor-sequence.svg' | relative_url }})

*Waiting on a test run. The agent works while the suite runs and a failure reaches it the moment it is printed. Every streamed line enters the context, so the monitor is filtered to the lines that change the next step. Diagram made with archify.*

### Seeing and managing background work

| | Claude Code | Codex |
|---|---|---|
| List background work | `/tasks` (shells and sub-agents, finished ones included) | `/ps` (background terminals and recent output) |
| Stop it | Ask the agent to stop it | `/stop` stops all background terminals |
| Whole session in the background | `/background`; watch all sessions with `claude agents` | |

<!-- IMAGE NEEDED: the Claude Code /tasks list (or Codex /ps) showing a running background shell and a finished sub-agent; search: "Claude Code background tasks", "Claude Code /tasks", "Codex CLI background terminals" -->

### Interaction with other features

| Feature | What a running background command does to it |
|---|---|
| `/goal` | Evaluation is skipped while background work runs; after 30 minutes a check-in asks the agent to read the output or fix stuck work |
| Resuming a session (`-c`, `-r`) | Background commands and monitors that were running are not restarted; ask the agent to start them again |
| Closing the session | Background commands and monitors stop |

## Monitor

### What it does

The Monitor tool watches a source and feeds each new line back to the agent as it arrives.

| Property | Detail |
|---|---|
| What it watches | A background command's output, or a file path |
| What the agent receives | Each new line, as a message, when it is printed |
| Polling | None; the agent keeps working until a line arrives |
| Lifetime | Stops when the session ends |
| Cost | Every delivered line enters the context |

### Waiting on a test run

```text
● Bash(python -m pytest tests -q > /tmp/run.log 2>&1; echo "EXIT $?" >> /tmp/run.log)
  run_in_background: true

● Monitor(command: tail -f /tmp/run.log | grep --line-buffered -E "FAILED|EXIT")
  ⎿  FAILED tests/test_sync.py::test_lock_release
  ⎿  EXIT 1
```

1. The full output goes to a file; nothing is lost.
2. The monitor passes only failure and exit lines to the agent.
3. The agent sees the failure as soon as it is printed and opens the test while the suite finishes.
4. It reads the full log only if the failure line is not enough.

### Patterns

| Pattern | Monitor command | Good for |
|---|---|---|
| Filtered log | `tail -f run.log \| grep --line-buffered -E "ERROR\|FAILED\|EXIT"` | Builds and test runs |
| Status file | `tail -f status.txt` where another process writes one line per state change | Deploys, long jobs |
| Directory watch | a loop that prints a line when an expected file appears | Waiting on another process's output |

### Trade-offs

| Trade-off | Detail | What to do |
|---|---|---|
| Context cost | A monitor on a chatty log fills the window with noise | Filter to the lines that change the next step; keep the full output in a file |
| Buffering | `grep` buffers output written to a pipe, so lines arrive late or in bursts | Use `grep --line-buffered` |
| Lifetime | A monitor ends with the session | For watching after you close the terminal, use a scheduled task or an external job |

| You are waiting on | Use |
|---|---|
| A command or sub-agent this session started, and only its result matters | Nothing extra: its completion wakes the session |
| Progress of that command before it finishes | A Monitor on its output |
| Something the session did not start, such as a pull request's checks or another process's file | A Monitor on a file the other process writes, or `/loop`; with Monitor available, a self-paced `/loop` may use it directly |

See [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}#loop).

## Headless runs

### claude -p

`claude -p "<prompt>"` runs one task without the interactive session and prints the result.

| In `-p` mode | Behaviour |
|---|---|
| Skills a person invokes | Work |
| Terminal-only built-in commands such as `/login` | Do not work |
| Context files and hooks | Load and run, unless `--bare` is set |
| A broken settings file | No dialog; Claude Code skips the file or entry and continues |
| `/goal` | Works; `claude -p "/goal <condition>"` runs to completion in one call |

### Flags for scripted runs

| Flag | What it does | Example |
|---|---|---|
| (pipe) | Sends content on standard input | `cat logs.txt \| claude -p "explain"` |
| `-c -p` | Continues the most recent conversation, non-interactively | `claude -c -p "Check for type errors"` |
| `--output-format` | `text`, `json` or `stream-json` | `claude -p "query" --output-format json` |
| `--json-schema` | Validated JSON output matching a schema | for a result another program parses |
| `--max-turns` | Limits agentic turns; no limit by default | `claude -p --max-turns 3 "query"` |
| `--allowedTools` | Tools that run without prompting | `--allowedTools "Read,Grep,Bash(npm test *)"` |
| `--disallowedTools` | Deny rules; a bare name removes the tool | `--disallowedTools "Edit,Write"` |
| `--tools` | Restricts the built-in tools | `--tools "Read,Grep,Glob"` |
| `--permission-mode` | Starts in a mode, such as `dontAsk` | `--permission-mode dontAsk` |
| `--bare` | Skips auto-discovery (below) | `claude --bare -p "query"` |

### --bare

`--bare` is a minimal mode: it skips auto-discovery so a job runs the same way on every machine, whatever is installed there.

| | `claude -p` | `claude -p --bare` |
|---|---|---|
| `CLAUDE.md` and auto memory | Loaded | Skipped |
| Hooks | Run | Skipped |
| Skills, custom commands, sub-agents, plugins | Available | Skipped, except skills in a directory passed with `--add-dir` |
| MCP servers | Connected | Skipped |
| Bash, file read and file edit tools | Available | Available |
| Use for | Scripted work that should follow the project's rules | Evaluations and read-only jobs that must not depend on the machine |

### A CI job

```yaml
- name: Summarise the failing tests
  run: >
    claude -p --bare --tools "Read,Grep,Glob" --max-turns 10
    "Read test-results.xml and list each failing test with a one-line cause" > summary.txt
```

- `--bare`: the same behaviour on every runner.
- `--tools "Read,Grep,Glob"`: the job can read and cannot edit or run commands.
- `--max-turns 10`: a bound on cost.

### A project-aware script

```bash
# Uses the project's CLAUDE.md and hooks; the tool flags enforce the scope
claude -p --allowedTools "Read,Grep,Edit(./CHANGELOG.md),Bash(git log *)" \
  --permission-mode dontAsk \
  "Update CHANGELOG.md from the commits since the last tag."
```

1. The prompt says what to do.
2. `--allowedTools` lets the job read, run `git log` and edit one file without prompting.
3. `--permission-mode dontAsk` denies anything else that would have prompted, so the job cannot wander into other files.
4. The project's hooks still run.

The scope is enforced by the flags; a sentence such as "do not edit any other file" in the prompt would only ask.

<details markdown="1">
<summary>Choosing between the two for an evaluation</summary>

- An evaluation compares runs. If the runs load whatever context files and hooks are on the machine, a difference in the result can come from the setup as easily as from the change under test.
- `--bare` removes that variable.
- When the thing under test is the harness itself, run without `--bare` and pin the harness version instead.

See [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}).

</details>

## Pitfalls and lessons

| Pitfall | What to do |
|---|---|
| An unfiltered monitor fills the context with log noise | Filter to failures and exit codes |
| A headless run hides settings errors | Run `claude doctor` afterwards |
| `--bare` drops the hooks your safety rules depend on | Use it for read-only jobs, or restrict the tools |
| A scope stated only in the prompt | Enforce it with `--allowedTools`, `--tools` or a permission mode |
| A long `-p` goal looks stuck | Add `--output-format stream-json --verbose` |
| Background commands vanish after a resume | Ask the agent to start them again |

## Sources

- Monitor tool and background commands: [Claude Code: tools reference](https://code.claude.com/docs/en/tools-reference), verified 2026-11-02.
- CLI flags, `--bare` and print mode: [Claude Code: CLI reference](https://code.claude.com/docs/en/cli-reference.md), read 2026-11-09; [Claude Code: headless mode](https://code.claude.com/docs/en/headless.md), verified 2026-11-09.
- `/goal` in `-p` mode and background work during a goal: [Claude Code: keep Claude working toward a goal](https://code.claude.com/docs/en/goal.md), read 2026-11-09.
- Background commands not restarted on resume: [Claude Code: run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks.md), read 2026-11-09.
- Settings errors in `-p` runs: [Claude Code: settings files and precedence](https://code.claude.com/docs/en/settings.md), read 2026-11-09.
- Codex `/ps` and `/stop`: [Codex: CLI slash commands](https://learn.chatgpt.com/docs/developer-commands.md?surface=cli), read 2026-11-09.
- Commands and scripts are illustrations written for this page.

{% include page_nav.html %}
