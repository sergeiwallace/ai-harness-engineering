---
title: "Goal and loop commands"
parent: "Native commands"
grand_parent: "Claude Code and Codex basics"
nav_order: 1
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# Goal and loop commands

`/goal` and `/loop` keep a session going without you typing the next prompt. `/goal` keeps one task going until a condition holds; `/loop` re-runs a prompt over time. They fail in different ways, and longer autonomous runs in later sections are built on them.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Choosing one

| What should start the next turn | Use | Stops when |
|---|---|---|
| The previous turn finishing, until a condition holds | `/goal` | A model judges the condition met or impossible, an error you must fix, or `/goal clear` |
| A time interval | `/loop` | You stop it, Claude decides the work is done (self-paced), or 7 days pass |
| The previous turn finishing, judged by your own script | A Stop hook in settings | Your script decides |
| A command or sub-agent this session started finishing | Nothing extra: its result wakes the session; add a [Monitor]({{ '/sections/03-claude-code-and-codex/background-work-and-headless-runs.html' | relative_url }}#monitor) to react to lines before then | The command ends |
| A schedule, with no open session | A cloud scheduled task (`/schedule`) or a desktop scheduled task | You delete it |

## Goal

### What it is

`/goal <condition>` sets a completion condition, and Claude keeps working toward it without a prompt per step.

- After each turn, a separate small model checks whether the condition holds.
- It is a session-scoped, prompt-based Stop hook: the same mechanism as a Stop hook in settings, active for this session only.
- One goal can be active per session; a new `/goal <condition>` replaces it.
- Codex has its own `/goal` (set, edit, pause, resume, view, clear).

![Sequence of two goal checks: a stop with a claim is refused because no test output follows the edit; after the agent runs the named test, the suite and the commit, the second stop is accepted]({{ '/assets/images/03-claude-code-and-codex/goal-checks-sequence.svg' | relative_url }})

*Two goal checks for one task. The evaluator answers each stop from the transcript alone. Diagram made with archify; the task is an illustration.*

### The commands

| You type | What happens |
|---|---|
| `/goal all tests in test/auth pass and lint is clean` | Sets the goal and starts a turn at once, with the condition as the directive |
| `/goal` | Shows the condition, how long it has run, turns evaluated, token spend and the evaluator's latest reason |
| `/goal clear` | Removes the goal; `stop`, `off`, `reset`, `none` and `cancel` also work |
| `/clear` | Starts a new conversation and removes any goal |
| `claude -p "/goal CHANGELOG.md has an entry for every PR merged this week"` | Runs the goal to completion in one non-interactive call |

<!-- IMAGE NEEDED: a Claude Code session with an active goal, showing the "/goal active" indicator and an evaluator verdict in the transcript; search: "Claude Code /goal", "Claude Code goal active indicator" -->

- While a goal is active, an `◎ /goal active` indicator shows how long it has run, and each verdict appears in the transcript; `Ctrl+O` shows the reason.
- In `-p` mode, add `--output-format stream-json --verbose` to see each turn; with plain text output a long goal can look stuck.
- Resuming a session (`-c`, `-r`) restores an active goal, with the turn count, timer and token baseline reset.

### How evaluation works

1. Claude finishes a turn.
2. Claude Code sends the condition and the conversation so far to the small fast model.
3. The model returns one of three verdicts, each with a reason:
   - Not yet met: Claude starts another turn and takes the reason as guidance.
   - Met: the goal clears and an achieved entry is recorded.
   - Impossible: the goal clears and a failed entry is recorded with the reason.
4. If Claude keeps answering the evaluator without using any tool for several turns, Claude Code stops the loop, warns you and leaves the goal set.

| Situation | What Claude Code does |
|---|---|
| A sub-agent or background shell is still running when a turn ends | Skips evaluation for that turn; the result arrives as a new turn |
| Background work keeps the goal waiting for 30 minutes | A check-in asks Claude to read the output, wait, or fix stuck work; later check-ins back off |
| Authentication failure, exhausted credits, a context overflow compaction could not clear, an unavailable model | Clears the goal and tells you to fix the cause and run `/goal` again |
| An overloaded server or a dropped connection | Retries, up to three times, then pauses |
| A rate limit, a usage limit or a hook that ended the turn | Pauses the goal and names the cause |

`CLAUDE_CODE_GOAL_CHECKIN_MINUTES` changes the 30-minute check-in interval; `0` turns check-ins and automatic retries off.

### The evaluator

#### What it can see

The evaluator has no tools. It reads the conversation and nothing else; it cannot run the tests or open a file to check a claim.

#### What that means for conditions

| Condition | Evaluator can judge it | Why |
|---|---|---|
| "the code is good" | No | No evidence in the conversation can settle it |
| "the tests pass" | Partly | It sees a test run if the agent printed one, but not which run is current |
| "`pytest tests/api` prints 0 failed after the last edit, and `git log -1` shows the fix commit" | Yes | Names the command, the output and the order |

#### Writing a good condition

1. One measurable end state: a test result, an exit code, a file count, an empty queue.
2. A stated check: how Claude proves it, such as "`npm test` exits 0" or "`git status` is clean".
3. The timing: the proof comes after the last change.
4. Constraints that matter: what must not change on the way, such as "no other test file is modified".
5. A bound: "or stop after 20 turns", which Claude reports against each turn.

The condition can be up to 4,000 characters.

### Two conditions for the same task

```text
Weak:   /goal fix the login bug
Strong: /goal tests/test_login.py::test_expired_token passes after the last edit,
        the full tests/auth suite prints 0 failed, no other test file is modified,
        and the fix is committed; or stop after 25 turns
```

| | Weak | Strong |
|---|---|---|
| What ends it | The agent saying "fixed" after a plausible edit | Two printed test runs after the last edit, and a commit |
| Can the evaluator contradict a false claim | No | Yes |
| Bound | None | 25 turns |

### Failure modes

| Failure | What happens | Fix |
|---|---|---|
| Stopping too early | The agent prints a claim ("all tests pass") without the output, and a lenient reading accepts it | Require the command output in the condition |
| Stopping on old evidence | A green run from before the last edit is still in the conversation | "after the last edit" in the condition |
| Running long on an unreachable target | A test depends on a service that is down; turns accumulate before the evaluator judges it impossible | Add a turn bound and an exit clause: "or, if the service is unreachable, report the error" |
| Waiting on prompts | The goal does not change the permission mode, so Manual mode still asks before tool calls | Run the goal in auto mode for unattended work, or allow the commands it needs |
| The command refuses to start | `/goal` follows the workspace trust rule for hooks, and is off when `disableAllHooks` is set or managed settings allow managed hooks only | Trust the folder, or check the hook settings |

<details markdown="1">
<summary>A goal and a Stop hook you write yourself</summary>

| | `/goal` | Stop hook in settings |
|---|---|---|
| Scope | This session | Every session in the file's scope |
| Check | A model reads the conversation | A script (deterministic) or a prompt (model-evaluated) |
| Typical use | One task, today | A rule that every session must meet before it stops, such as "tests pass" |

For a check that must hold every time, a Stop hook running a script is the enforced form; `/goal` is the one-line version for one task. See [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

</details>

### Model, cost and availability

| Item | Detail |
|---|---|
| Evaluator model | Your configured small fast model; `ANTHROPIC_DEFAULT_HAIKU_MODEL` changes it (and every other use of that model) |
| Cost | Evaluation tokens are billed on that model and are typically small next to the main turns |
| Provider | Runs on the session's provider, cloud providers such as Amazon Bedrock included |
| Codex | Its own `/goal` |

## Loop

### What it is

`/loop` re-runs a prompt on repeat while the session stays open.

- It is a bundled skill: Claude turns the interval into a cron schedule and fires the prompt between your turns.
- Typical uses: poll a deployment, watch a pull request, check back on a long build.
- Codex has no `/loop`.

### The three forms

| You provide | Example | What happens |
|---|---|---|
| Interval and prompt | `/loop 5m check the deploy` | Runs on a fixed schedule |
| Prompt only | `/loop check whether CI passed and address any review comments` | Claude picks the next delay each time, between 1 minute and 1 hour, and prints it with the reason |
| Interval only, or nothing | `/loop` | Runs a built-in maintenance prompt, or your `loop.md` |

- Interval units are `s`, `m`, `h` and `d`, as a leading token (`30m`) or a trailing clause (`every 2 hours`).
- Seconds round up to a minute, and intervals that do not map to a clean cron step (`7m`, `90m`) are rounded; Claude says what it picked.
- A skill can be the prompt: `/loop 20m /review-pr 1234`. Built-in commands and skills marked `disable-model-invocation: true` arrive as plain text instead.
- With the Monitor tool available, Claude may use Monitor for a self-paced loop, which avoids polling.

### The maintenance prompt and loop.md

A bare `/loop` works through, in order: unfinished work from the conversation; the current branch's pull request (review comments, failed CI, conflicts); then cleanup passes. It does not start new work, and pushes or deletions proceed only when they continue something the transcript already authorized.

To replace it, write `.claude/loop.md` (project, takes precedence) or `~/.claude/loop.md` (user):

```markdown
Check the `release/next` PR. If CI is red, pull the failing job log,
diagnose, and push a minimal fix. If new review comments have arrived,
address each one and resolve the thread. If everything is green and
quiet, say so in one line.
```

Edits apply on the next iteration; content past 25,000 bytes is truncated.

### Stopping a loop

| Loop | How to stop it |
|---|---|
| Self-paced | `Esc` while it waits for the next iteration, or `Ctrl+C` while Claude is idle; Claude can also end it once the task is done |
| Fixed interval | Ask Claude to cancel it: "cancel the deploy check loop"; `Esc` and `Ctrl+C` do not stop it |
| Any | It ends 7 days after creation, after one final run |

Ask "what scheduled tasks do I have?" to list them. A session holds up to 50 scheduled tasks.

### Limits

- Tasks fire only while Claude Code is running and idle, between your turns; closing the terminal stops them. Backgrounding the session carries them over.
- There is no catch-up: a fire missed during a long turn runs once when Claude becomes idle.
- Run times are shifted slightly (jitter): a recurring task runs up to half its interval late, at most 30 minutes; `/loop 5m` runs every 4 minutes 45 seconds.
- On resume, fixed-interval tasks are restored if not expired; a self-paced loop is not, so run `/loop` again.
- `CLAUDE_CODE_DISABLE_CRON=1` turns the scheduler and `/loop` off.

### Costs

Each iteration is a turn, and each turn adds to the context.

- A loop on a fast interval with a verbose prompt fills the window.
- Ask for a short status line in the prompt:

```text
> /loop 10m check the staging deploy; reply with one line: HEALTHY, FAILED <reason>, or PENDING
```

- Stop the loop once its question is answered, or compact with a focus if it must keep going.

### Loop and the scheduling options

| | `/loop` | Desktop scheduled task | Cloud scheduled task |
|---|---|---|---|
| Runs on | Your machine, in the session | Your machine | The cloud, on a fresh clone |
| Needs an open session | Yes | No | No |
| Needs the machine on | Yes | Yes | No |
| Local files | Yes | Yes | No |
| Minimum interval | 1 minute | 1 minute when Claude sets it, 1 hour in the forms | 1 hour |

## Goal and loop side by side

| | `/goal` | `/loop` |
|---|---|---|
| Ends when | The condition is judged met or impossible | You stop it, Claude ends a self-paced loop, or 7 days pass |
| Checked by | A small evaluator model | Nobody; it repeats |
| Typical use | Fix, build or migrate until done | Watch something external |
| Main risk | A weak condition stops early | Context fills with repeated turns |

## Sources

- `/goal`: [Claude Code: keep Claude working toward a goal](https://code.claude.com/docs/en/goal.md), read 2026-11-09.
- `/loop`, scheduling options, limits and jitter: [Claude Code: run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks.md), read 2026-11-09.
- Codex `/goal` and the absence of `/loop`: [Codex: CLI slash commands](https://learn.chatgpt.com/docs/developer-commands.md?surface=cli), read 2026-11-09.
- The example conditions and the failure-mode table are written for this page from those docs.

{% include page_nav.html %}
