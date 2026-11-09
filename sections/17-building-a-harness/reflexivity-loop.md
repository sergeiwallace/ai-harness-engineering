---
title: "The improvement loop"
parent: "Building your own harness"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/17-building-a-harness/detail.html
  - /sections/17-building-a-harness/advanced.html
last_updated: "2026-11-09"
talk_stop: "12"
stop_minutes: "5"
---

{% include page_nav.html %}

# The improvement loop

The improvement loop is the [systems approach]({{ '/sections/17-building-a-harness/' | relative_url }}#1-a-systems-approach-to-building-a-harness) applied again to each gap the agents expose. Each friction point becomes a change to the system, enforced where possible by a workflow or tool, a hook, script or linter, or a test, and the loop keeps the change only if it helped.

- Input: one friction point, such as a failure, a stall or a repeated correction.
- Output: one change to the system, measured on the next sessions.
  - The change sits at layer 2, 3 or 4 where it can.
  - A context-file line, if any, names the guard as a second layer.

![Sequence diagram of the improvement loop on the tests-skipped task: friction, record, find the part, change the harness with a Stop hook, then measure on the next task, where the hook blocks an untested finish and the agent fixes a real failure.]({{ '/assets/images/17-building-a-harness/reflexivity-loop-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. The five steps on one real task, which the [worked example](#a-full-worked-example) below walks through in full. Diagram made with archify.*

## The five steps

### 1. Friction

Friction is any of these:

| Kind | What it looks like |
|---|---|
| Failure | The agent produced something wrong and said it was done |
| Stall | The agent looped, waited, or asked a question it could have answered itself |
| Repeated correction | You told it the same thing in two different sessions |
| Your own repetition | You typed the same long prompt again |

The last kind is easy to miss because it does not look like a failure. It costs you time on every run, and a [skill]({{ '/sections/05-skills-and-plugins/' | relative_url }}) removes it.

#### When friction is worth a harness change

Rule of thumb: if you have typed the same correction or the same prompt twice, consider building it into the harness. Use judgement as well, and fix it the first time when the context shows it will happen again, or when a repeat would be costly or risky.

| Trigger | Example | First fix |
|---|---|---|
| The second time you type the same correction or prompt | You ask for the same review steps in two sessions | A skill that runs those steps |
| The context shows it will recur | An agent hand-edits a generated file in a repository where most tasks touch the schema, so every later task can hit the same mistake | A PreToolUse hook that refuses edits under `generated/` and names the schema file |
| A repeat would be costly or risky | An agent force-pushes to a shared branch once and overwrites a colleague's commits | A permission rule that denies force pushes, added the same day |

In each case the fix goes into the system first: a workflow, a hook, a check or a permission rule. A context-file line, if you add one, only points at it.

### 2. Record

Write it down before fixing it. Each entry is short and dated, and a repeat raises the count on the existing entry.

| Where to keep the log | Suits | Watch for |
|---|---|---|
| A text file in the repository | One person, one project | Merge conflicts if several sessions write to it |
| An issue tracker | Several projects, or agents filing their own entries | Duplicates; search before filing |
| A label on existing issues | A team that already tracks bugs there | Friction that is not a bug, such as a retyped prompt, gets left out |

What each entry needs:

- The date, and how often it has happened so far.
- What happened, and what you expected instead.
- A pointer to the transcript or the output, so step 3 can read it.

<details markdown="1">
<summary>Why record before fixing</summary>

A fix made in the moment is shaped by the last failure. A fix made after a week of records is shaped by the pattern. Recording also stops you from fixing a one-off that will never happen again, which is how complexity creep starts (see [Complexity creep and pruning]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }})). When you do fix something the first time, because it will recur or a repeat would be risky, record it anyway, so the log shows why the mechanism exists.

</details>

### 3. Find the part that let it through

Ask which part of the system was missing or unclear. A useful order is to ask, in turn:

1. Did the agent have the information?
   - If not, ask first whether a workflow step or a check can supply it (a script that reads the setting, a template field). Use a context-file line only for knowledge no check can encode.
2. Did it have the information and ignore it?
   - Then the gap is a check: a hook, a linter or a test.
3. Was the content right and the shape wrong?
   - Then the gap is a template or a validator for the output's format.
4. Was a step slow or manual?
   - Then the gap is a tool or a skill.
5. Was it an action you would have refused?
   - Then the gap is an approval rule.

Read the transcript as well as the result. The transcript usually shows the exact step where the agent took the wrong branch, and whether a rule was in its context at that moment.

### 4. Change the system

Make the smallest change that would have stopped this case, at a layer that enforces it:

1. A workflow or tool that builds the right step in (layer 2).
2. A hook, script or linter that refuses or flags the action as it happens (layer 3).
3. A test or validation that catches the result before it lands (layer 4).
4. Then, at most, one context-file line that names the guard (layer 1, the second line of defence).

The ladder, from soft to hard:

| Rung | The rule lives in | It holds when |
|---|---|---|
| 1 | An instruction in a context file | The agent remembers and chooses to follow it |
| 2 | A skill or template | The agent loads it for this task |
| 3 | A script or linter | Someone runs it |
| 4 | A hook | Always, on every matching event |
| 5 | A test or merge gate | Always, before the result lands |
| 6 | A permission rule | Always; the action is unreachable |

![Sequence diagram of a never-force-push rule moving from an instruction, which the agent breaks, to a PreToolUse hook that refuses with a next step, to a permission rule that denies the command before it runs.]({{ '/assets/images/17-building-a-harness/enforcement-ladder-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 2. One rule moving down three rungs. Diagram made with archify.*

Start at the softest step that works. Move down when agents keep breaking the rule. [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers each step's mechanics.

### 5. Measure, then keep or revert

Treat the change as a trial. Check it against the next sessions, or against a small task set made from past failures (see [Building a task set]({{ '/sections/18-trends/building-a-task-set.html' | relative_url }}) and [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }})). Keep it if the friction stops and nothing else got worse. Revert it if it blocked correct work or did not change the outcome.

| Result after a week | Decision |
|---|---|
| Friction gone, no false blocks | Keep |
| Friction gone, some false blocks | Keep, narrow the rule |
| Friction still there | Revert, look again at step 3 |
| Friction gone, but another failure appeared | Revert, then fix both together |

## A full worked example

This is the "forgot to run the tests before saying done" entry from the [overview's friction log]({{ '/sections/17-building-a-harness/' | relative_url }}#21-keep-a-friction-log), taken through all five steps. The example is illustrative: the failure is a common one, and the dates, counts and file names are made up for this page.

| Step | Outcome |
|---|---|
| Friction and record | Four "done" endings with a failing test in one week |
| Find the part | A missing check at layer 3 |
| Change | A `Stop` hook, plus a one-line pointer in `CLAUDE.md` |
| Measure | Five fires in the next week, all on real failures; no false blocks |

### Friction and record

```text
2026-11-03  agent said done with a failing test   seen 4   cost medium
  sessions: Mon (parse_date), Tue (config loader), Thu twice (CLI flags)
  caught by: my review each time
```

### Find the part

1. Was the rule in context?
   - Yes. `CLAUDE.md` already said "Run the tests before you finish."
2. Did the agent read it?
   - Yes. Each transcript shows a test run early in the task.
3. Where did it go wrong?
   - After the test run it made one more edit and did not run the tests again.
4. Which part was missing?
   - A check at layer 3. Another context-file line would sit on the same layer that had already failed four times.

### Change: a Stop hook

How the pieces act together at the end of a turn (Figure 1, step 5):

1. The agent tries to end its turn, and Claude Code runs the `Stop` hook.
2. The hook compares two times.
   - The time of the last passing test run, from a stamp file that the test wrapper touches on success.
   - The time of the newest edit in the working tree.
3. If an edit is newer, the hook exits with code 2.
   - Claude Code keeps the agent working and passes the hook's message back to it.
   - The message names the next step: run the tests and fix any failure.
4. The agent runs the tests. On a pass, the stamp is newer than every edit, and the next attempt to finish succeeds.

```bash
#!/usr/bin/env bash
# Stop hook: refuse to finish while edits are newer than the last passing test run.
set -euo pipefail
stamp=".test-pass-stamp"          # the test wrapper touches this file on success
newest_edit=$(git ls-files -m -o --exclude-standard | xargs -r stat -c %Y 2>/dev/null | sort -n | tail -1 || true)
last_pass=$(stat -c %Y "$stamp" 2>/dev/null || echo 0)
if [[ -n "${newest_edit:-}" && "$newest_edit" -gt "$last_pass" ]]; then
  echo "Tests have not passed since your last edit. Run the test command, fix any failure, then finish." >&2
  exit 2
fi
exit 0
```

The settings entry that registers it:

```json
{
  "hooks": {
    "Stop": [
      { "hooks": [ { "type": "command", "command": "$CLAUDE_PROJECT_DIR/tools/require-fresh-tests.sh", "timeout": 10 } ] }
    ]
  }
}
```

LangChain used the same idea in its own agent: a `PreCompletionChecklistMiddleware` that "intercepts the agent before it exits" and asks for a verification pass. It was one of the harness changes behind a measured gain on Terminal Bench 2.0, from 52.8% to 66.5% with the model unchanged (see the [overview]({{ '/sections/17-building-a-harness/' | relative_url }}#43-what-the-research-says)).

The hook also receives a `stop_hook_active` field on its input when the agent is already continuing because of a Stop hook; a production version reads it so that a test suite that cannot pass does not trap the session in a loop.

Then shorten the line in `CLAUDE.md` to "Tests must pass after your last edit; a Stop hook enforces this." The context file now names the guard that enforces the rule.

### Measure

Read the next week against the decision table in step 5:

| Question | Answer in this example | Decision |
|---|---|---|
| Did the friction stop? | No "done with a failing test" endings | Keep |
| Did the hook fire on real failures? | Five times, each followed by a test run and a fix | Keep |
| Did it block a correct finish? | Never | No narrowing needed |

## Who runs the loop

Early on, you do. Later, agents can run most of it:

| Step | Early | Later |
|---|---|---|
| Friction | You notice it | Agents report it in a fixed format |
| Record | You write the log | Agents file issues |
| Find the part | You read the transcript | An agent reads it and proposes the part |
| Change | You write it | An agent writes the change and its test |
| Measure | You watch | An evaluation run compares before and after |
| Keep or revert | You decide | You still decide |

The last row stays with a person. The research reports the same split: the evaluator and the final decision stay outside the thing being changed. See [Self-improving harness research]({{ '/sections/17-building-a-harness/self-improving-harnesses.html' | relative_url }}).

## Examples from my own harness

- Guards that started as written rules.
  - The pattern matched the worked example: the rule was in context, the transcript showed the agent had read it, and the fix was a check that runs without the agent's cooperation.
- Sessions that report their own friction.
  - When a session hits a blocker the harness should have prevented, it files an issue with the transcript excerpt.
  - A later session fixes the harness and adds a regression test for the fix.

To start: pick the item at the top of your friction log and take it through all five steps, including the measure step.

## Pitfalls

<details markdown="1">
<summary>Fixing the symptom in the prompt</summary>

Adding "IMPORTANT: always run the tests" in capital letters keeps the rule on the softest rung. If the agent already had the rule, more emphasis rarely helps for long. Move to a check.

</details>

## Sources

- Claude Code hooks, the `Stop` event and exit code 2: [Claude Code docs: Hooks](https://code.claude.com/docs/en/hooks.md), not re-checked for this page.
- LangChain's verification middleware and Terminal Bench 2.0 result: [LangChain, Improving Deep Agents with harness engineering](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering), verified 2026-11-09.
- From the author's own harness, described conceptually.

{% include page_nav.html %}
