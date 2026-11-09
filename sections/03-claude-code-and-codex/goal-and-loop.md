---
title: "Goal and loop commands"
parent: "Native commands"
grand_parent: "Claude Code and Codex basics"
nav_order: 1
---

{% include page_nav.html %}

# Goal and loop commands

As of 2026-11-09. Sub-topic of [Native commands]({{ '/sections/03-claude-code-and-codex/native-commands.html' | relative_url }}) in [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}). Slide: [stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

`/goal` and `/loop` are the two native commands that keep a session going without you typing the next prompt. `/goal` keeps one task going until it is done. `/loop` repeats a prompt over time. They solve different problems, fail in different ways, and are the base that longer autonomous runs in later sections are built on.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Goal

### What it is

`/goal <condition>` keeps the session working until a separate small model judges the condition met. Claude Code implements it as a session-scoped, prompt-based Stop hook: every time the agent tries to end its turn, the hook runs the evaluator, and the evaluator either lets the stop through or sends the agent back with the reason. Codex has a `/goal` command too.

![Sequence of two goal checks: a stop with a claim is refused because no test output follows the edit; after the agent runs the named test, the suite and the commit, the second stop is accepted]({{ '/assets/images/03-claude-code-and-codex/goal-checks-sequence.svg' | relative_url }})

*Two goal checks for one task. The evaluator answers each stop from the transcript alone. Diagram made with archify; the task is an illustration.*

### The evaluator

#### What it can see

The evaluator has no tools. It reads the conversation and nothing else. It cannot run the tests or open a file to check a claim; it can only judge what the agent printed.

#### What that means for conditions

| Condition | Evaluator can judge it | Why |
|---|---|---|
| "the code is good" | No | No evidence in the conversation can settle it |
| "the tests pass" | Partly | It sees a test run if the agent printed one, but not which run is current |
| "`pytest tests/api` prints 0 failed after the last edit, and `git log -1` shows the fix commit" | Yes | Names the command, the output and the order |

#### Writing a good condition

1. Name the command whose output proves it.
2. Name the output the evaluator should see.
3. Say when, relative to the last change, the proof must come.
4. Add the end state you want, such as a commit or a pushed branch.

### Two conditions for the same task

```text
Weak:   /goal fix the login bug
Strong: /goal tests/test_login.py::test_expired_token passes after the last edit,
        the full tests/auth suite prints 0 failed, and the fix is committed
```

With the weak condition, the agent can say "fixed" after a plausible edit, and the evaluator has nothing to contradict it. With the strong one, the agent has to run and show both test commands after its final edit.

### Failure modes

#### Stopping too early

The agent prints a claim ("all tests pass") without the output, and a lenient reading accepts it. Fix: require the command output in the condition.

#### Never stopping

The condition can never be met, for example because a test depends on a service that is down. The agent loops. Fix: add an exit clause, such as "or, if the service is unreachable, the agent has written down the error and stopped".

#### Stopping on the wrong evidence

An old green run from before the last edit is still in the conversation. Fix: "after the last edit" in the condition.

<details markdown="1">
<summary>How the goal hook relates to hooks you write</summary>

A goal is a Stop hook the client installs for the session, with a prompt handler. You can write the same shape yourself in `settings.json` as a Stop hook with a `prompt` handler, and make it permanent for a project. The `/goal` command is the session-scoped, one-line version. See [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

</details>

### Availability

`/goal` works on cloud providers such as Amazon Bedrock; the evaluator runs on the session's provider.

## Loop

### What it is

`/loop` repeats a prompt on an interval, or at a pace the agent sets itself. Recurring tasks expire after 7 days. Codex has no `/loop`.

```text
> /loop 5m check the deploy status of the staging release and tell me
  when it is healthy or has failed
```

### When to use it

| Good fit | Poor fit |
|---|---|
| Polling something outside the session: a deploy, a queue, a pull request's checks | Waiting for a command the session started; use a [background command and Monitor]({{ '/sections/03-claude-code-and-codex/background-work-and-headless-runs.html' | relative_url }}) |
| A recurring check while you are in the session | A job that must run with the machine off; use `/schedule` |
| Self-paced work where the agent decides when to look again | A single task with an end state; use `/goal` |

### Costs

Each iteration is a turn, and each turn adds to the context. A loop on a fast interval with a verbose prompt fills the window. Make the prompt ask for a short status line, and compact or stop the loop once its question is answered.

## Goal and loop side by side

### Choosing one

1. Does the work have an end state you can name? Use `/goal` with a condition that names its evidence.
2. Is it the same check over time, while you are in the session? Use `/loop`.
3. Must it run with the machine off? Use `/schedule`.
4. Is it waiting on a command this session started? Use a background command and [Monitor]({{ '/sections/03-claude-code-and-codex/background-work-and-headless-runs.html' | relative_url }}#monitor).

### Comparison

A goal ends on a judgement; a loop ends on a timer or when you stop it.

| | `/goal` | `/loop` |
|---|---|---|
| Ends when | The condition is judged met | You stop it, or after 7 days |
| Checked by | A small evaluator model | Nobody; it repeats |
| Typical use | Fix, build or migrate until done | Watch something external |
| Main risk | Weak condition stops early | Context fills with repeated turns |

## Sources

- `/goal`: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-09.
- `/loop` and scheduled tasks: <https://code.claude.com/docs/en/scheduled-tasks.md>, verified 2026-11-08.
- Codex `/goal` and the absence of `/loop`: <https://learn.chatgpt.com/docs/developer-commands.md?surface=cli>, verified 2026-11-08.
- The conditions, failure modes and examples are written for this page from those docs and from use.

{% include page_nav.html %}
