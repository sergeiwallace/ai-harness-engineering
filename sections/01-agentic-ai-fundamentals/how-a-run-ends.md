---
title: "How an agent run ends"
parent: "The agent loop"
grand_parent: "Agentic AI fundamentals"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# How an agent run ends

An agent run ends in one of four ways: the model says it is done, a person stops it, a limit fires, or a check decides it is not done and sends it back. Two stopping problems come up again and again in long autonomous work:

- a run that ends too early with a confident "done"
- a run that never ends, because the goal cannot be reached and nothing stops it.

This page is part of [The agent loop]({{ '/sections/01-agentic-ai-fundamentals/the-agent-loop.html' | relative_url }}). It covers each exit, the Claude Code settings and commands behind it, and a checklist for long runs.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The four exits

![Diagram: the paths out of an agent run, top to bottom. A task starts the working loop; from working, Esc leads to interrupted and a cap leads to limit reached; when the model judges it done it writes a final answer and tries to stop; a completion check either confirms the condition and the run is done, sends it back to working with a reason, or after its retry cap ends in a blocked report]({{ '/assets/images/01-agentic-ai-fundamentals/run-end-paths.svg' | relative_url }}){: style="width: 100%" }
*Figure: every way a run can end. The model chooses the final answer; the completion check, the limits and you decide whether the run actually stops there.*

| Exit | Who decides | Typical trigger | Claude Code mechanism | Risk |
|---|---|---|---|---|
| Final answer | the model | it judges the goal met, or decides it is blocked | a reply with stop reason `end_turn` | stops early with work undone |
| Interrupt | a person | the run is heading the wrong way | Esc in a session, Ctrl+C for `claude -p` | needs someone watching |
| Limit | the client or a wrapper | turns, money or time run out | `--max-turns`, `--max-budget-usd`, timeouts | stops mid-step, state may be half-written |
| Completion check | a hook, an evaluator or a script | the stated condition is not yet true | `Stop` hooks, `/goal` | can loop if the condition is unreachable |

## Exit 1: the final answer

### What it is

When the model writes a reply with no tool call, the client treats it as the end of the task.

| Stop reason in the reply | What the client does |
|---|---|
| `tool_use` | runs the tools and calls the model again |
| `end_turn` | treats the text as the final answer, unless a completion check sends it back |

[Messages and content blocks]({{ '/sections/01-agentic-ai-fundamentals/the-agent-loop.html' | relative_url }}#stop-reasons) lists all seven stop reasons.

### Why it can be wrong

| Failure | What happens | Example |
|---|---|---|
| Premature completion | the model's "done" rests on what is in its context, which may be partial | it ran one test, saw it pass, and reported the task complete without running the suite |
| False report | the final answer is prose about its own work, which can be wrong | it says "all tests pass" after a compaction dropped the record of what was run |

### Worked example: reporting a blocked task

An agent asked to deploy a service finds the cloud credentials expired.

| | Final answer | What happened |
|---|---|---|
| Good | "Blocked: the deploy step needs a fresh login; nothing was deployed." | states what stopped it and what state things are in |
| Bad | "Deployment steps complete." | every command it could run did run, so it reported success |

Two layers make the good answer the usual one:

1. A completion check that tests the outcome, such as the service's health endpoint returning the new version, so a false "complete" is sent back.
2. An instruction in the context file or brief that "blocked" is an acceptable answer, with the fields a blocked report must contain: what failed, what was done, what is needed.

## Exit 2: the interrupt

A person can stop a run at any moment.

| Where | Key | What happens |
|---|---|---|
| An interactive session | Esc | the current turn stops; the conversation stays open |
| A `claude -p` run | Ctrl+C | the process ends |

After Esc in a session:

1. Any partial work, such as an edit already applied, stays in place.
2. The model sees the interruption in its context.
3. You type a correction ("use the existing helper instead") and the run continues from there.
4. If the partial work is wrong, `/rewind` rolls the conversation and the code back to an earlier checkpoint.

Interrupts need someone watching. For long unattended runs, limits and completion checks do the same job without a person at the screen.

## Exit 3: limits

### Kinds of limit

| Limit | Claude Code setting | Notes |
|---|---|---|
| Turns, for a headless run | `claude -p --max-turns 50` | print mode only; no limit by default |
| Money, for a headless run | `claude -p --max-budget-usd 5` | print mode only; checked against Claude Code's own cost estimate |
| Turns, for a sub-agent | `maxTurns` in the sub-agent's definition | the result is marked partial and can be resumed |
| One command's run time | `BASH_DEFAULT_TIMEOUT_MS`, `BASH_MAX_TIMEOUT_MS` | 2 minutes by default, 10 minutes at most |
| The whole job | the CI job's own timeout | covers everything, including hung processes |
| Context | the model's window, with `/autocompact` to set when compaction starts | compaction keeps the run going |

### The context limit is different

A full context window does not end a Claude Code session:

1. The window nears its limit.
2. The client compacts the history into a summary.
3. The run carries on from the summary, with less detail than it had.
   - The project-root `CLAUDE.md`, unscoped rules and auto memory are re-injected from disk.
   - Chat-only instructions survive only if the summary kept them.

[The context window and tokens]({{ '/sections/01-agentic-ai-fundamentals/context-window-and-tokens.html' | relative_url }}#compaction) covers what survives.

### Worked example: a limit with a clean stop

A nightly job runs an agent headless with a turn cap. When the cap fires mid-task, the run ends wherever it was, which can leave a half-edited file.

```bash
claude -p "Migrate the next table in PLAN.md. After 40 turns, stop starting new work, \
commit what is done, and update PLAN.md with what is left." \
  --max-turns 50 --max-budget-usd 5
```

| Layer | What it does |
|---|---|
| The instruction at 40 turns | usually produces a clean stop: work committed, plan updated |
| The caps at 50 turns and 5 dollars | end the run if the instruction is missed |
| A check at the start of the next run | reads `PLAN.md` and `git status`, and stops if the working tree is not clean |

## Exit 4: completion checks

### The idea

A completion check runs when the model tries to finish and tests the stated condition. The figure at the top of this page shows it as the "not yet" arrow.

1. The model writes a final answer: "done".
2. The client runs the check before ending the turn.
3. The check answers.
   - Condition met: the run ends.
   - Not met: the client returns the reason to the model ("2 tests still fail") and the loop continues.
4. The model works again, writes a new final answer, and the check runs again.
5. After the check's cap, the run stops and reports what blocks it.

### Three ways to build one

#### A Stop hook

Claude Code runs `Stop` hooks when the agent is about to end its turn. A hook can block the stop by exiting with code 2, or by printing JSON with a reason the model reads:

```json
{"decision": "block", "reason": "2 tests still fail in tests/test_dates.py; fix them before finishing."}
```

A minimal hook that runs the suite and guards against looping:

```bash
#!/bin/bash
INPUT=$(cat)
# second attempt in a row: let it stop, so the hook cannot loop forever
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ]; then
  exit 0
fi
if ! pytest -q > /tmp/stop-check.log 2>&1; then
  echo '{"decision": "block", "reason": "Tests fail; see the last run and fix them before finishing."}'
fi
exit 0
```

[Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) shows how hooks are registered.

#### An evaluator: `/goal`

`/goal` sets a condition in plain words, and Claude keeps working until it holds.

```text
/goal all tests in test/auth pass and the lint step is clean, or stop after 20 turns
```

1. After each turn, a small fast model reads the condition and the conversation.
2. It returns one of three verdicts, each with a short reason:
   - not yet met: Claude takes the reason as guidance and starts another turn
   - met: the goal clears and an achieved entry is recorded
   - impossible: the goal clears and a failed entry is recorded with the reason.
3. The evaluator does not run commands or read files. It judges only what Claude has shown in the conversation, so the condition must name a check Claude will run, such as "`npm test` exits 0".

| Command | What it does |
|---|---|
| `/goal <condition>` | sets the goal (up to 4,000 characters) and starts working at once |
| `/goal` | shows the condition, run time, turns evaluated, token spend and the latest reason |
| `/goal clear` | removes the goal; `/clear` also removes it |
| `claude -p "/goal <condition>"` | runs the loop to completion headless; add `--output-format stream-json --verbose` to watch it |

`/goal` itself is a prompt-based `Stop` hook for the current session. It does not change the permission mode, so an unattended goal usually runs in `auto` mode.

#### A wrapper script

A script outside the client runs the agent, checks the result itself (exit codes, files, test output), and starts another run with the failure if needed. It is the most transparent of the three and the easiest to log.

<details markdown="1">
<summary>Choosing between them</summary>

| | Stop hook | `/goal` | Wrapper script |
|---|---|---|---|
| Condition type | scriptable: tests, lint, a file exists | judged in words from the conversation | scriptable |
| Scope | every session the settings file covers | the current session only | the runs the script starts |
| Runs inside the session | yes | yes | no, it starts sessions |
| Cost | the check's run time | a small-model call per turn | a full new run per retry |
| Failure mode | a slow check slows every stop | a lenient judge passes weak work | each retry starts without the last session's context |

Which to use:

- a Stop hook fits anything a script can measure
- `/goal` fits conditions that need judgement, such as "the design doc answers every open question"
- in the [systems approach]({{ '/' | relative_url }}#a-systems-approach), the scripted check is the enforcing layer, and a `/goal` condition is closer to an instruction with a reader.
</details>

### The unreachable condition

A completion check can create the loop it was meant to prevent. If the condition cannot be met, for example because the tests need a service that is down, the agent is sent back again and again.

| Mechanism | Built-in stop |
|---|---|
| A `Stop` hook | Claude Code overrides it after 8 consecutive blocks with no tool call in between (`CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` changes the number); the hook should also check `stop_hook_active` |
| `/goal` | the evaluator can return "impossible"; the loop also stops if Claude answers several turns in a row without using a tool |
| A wrapper script | only the retry count you write into it |

## A stopping checklist for long runs

1. State the goal as a condition a check can test ("the suite passes", "the file exists"). An activity such as "work on the tests" gives a check nothing to test.
2. Say that "blocked" is an acceptable answer, and what a blocked report must contain.
3. Put a scriptable check at the stop step.
4. Cap the retries of that check.
5. Put a hard limit outside everything: turns, money or the job's timeout.
6. Make the agent write its state to a file before any planned stop, so the next run can resume (see [Sessions and state]({{ '/sections/01-agentic-ai-fundamentals/sessions-and-state.html' | relative_url }})).

## Sources

- `/goal`: syntax, evaluator, verdicts, status and clearing, headless use, and the no-progress stop: [Claude Code docs: Keep Claude working toward a goal](https://code.claude.com/docs/en/goal.md), verified 2026-11-09.
- `Stop` hook blocking, the 8-block cap, `stop_hook_active` and `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP`: [Claude Code docs: Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide.md) and [Claude Code docs: Hooks reference](https://code.claude.com/docs/en/hooks.md), verified 2026-11-09.
- `--max-turns` and `--max-budget-usd`: [Claude Code docs: CLI reference](https://code.claude.com/docs/en/cli-reference.md), verified 2026-11-09.
- `/rewind` and `/autocompact`: [Claude Code docs: Commands](https://code.claude.com/docs/en/commands.md) and [Claude Code docs: Explore the context window](https://code.claude.com/docs/en/context-window.md), verified 2026-11-09.
- Command timeouts and sub-agent `maxTurns`: [Claude Code docs: Tools reference](https://code.claude.com/docs/en/tools-reference.md), verified 2026-11-09.
- The examples, the hook script and the checklist are written for this page.

{% include page_nav.html %}
