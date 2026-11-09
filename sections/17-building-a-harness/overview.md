---
title: "Building your own harness"
nav_order: 27
has_children: true
has_toc: false
permalink: /sections/17-building-a-harness/
---

{% include page_nav.html %}

# Building your own harness

As of 2026-11-09. Overview page. Slides for this part of the talk: [Stop 12 on the talk page]({{ '/talk/' | relative_url }}#stop-12-5-minutes).

You build a harness with a systems approach: each time an agent's output falls short, you find the part of the system that let it through and change that part, so the fix is enforced on every later run. Repeating this for every gap is how a harness improves. This overview covers how to start, where each fix belongs, and how to keep the harness from growing into something nobody can follow. Each part links to a topic page that goes deeper.

![Sequence diagram of the improvement loop on one real task. The agent reads the rule to run tests, runs them early, makes one more edit and reports done; review finds a failing test; you record it and find that the rule was in context, so the gap is a check; the agent writes a Stop hook; on the next task the hook blocks a finish with untested edits, and the agent re-runs the tests and fixes a real failure.]({{ '/assets/images/17-building-a-harness/reflexivity-loop-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. The improvement loop on one real task: friction, record, find the part, change the system, then measure on the next task. Diagram made with archify.*

| If you read only one thing | Where it goes deeper |
|---|---|
| Treat each failure as a gap in the system, and change the system | [The improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}) |
| Enforce the fix with a workflow, a hook or a test; a context-file line is the second layer | [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) |
| Your first week can be four small files | [A first-week harness]({{ '/sections/17-building-a-harness/first-week.html' | relative_url }}) |
| A new context-file rule after every mistake is how complexity creep starts | [Complexity creep and pruning]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }}) |
| Agents can do most of the improvement work | [Self-improving harness research]({{ '/sections/17-building-a-harness/self-improving-harnesses.html' | relative_url }}) |

## 1. A systems approach to building a harness

The model and everything around it form one system. When the output is wrong, some part of that system let it through, and the improvement is a change to that part.

- The model, which you choose but do not change.
- Its context: context files, skills, prompts and the conversation so far.
- Its tools and workflows: the commands, scripts and procedures it runs.
- Its checks: hooks, linters, tests, gates and permission rules.
- You: the reviewer, and the person who approves outward and destructive actions.

### 1.1 The four layers

A harness has four layers. The first guides the model; the other three enforce.

| Layer | What it contains | How it acts | Reliability over a long session |
|---|---|---|---|
| 1. Instructions | Context files, prompts, and the guidance text in skills | Guides the model while it reads them | Applied inconsistently when the model juggles several things, switches tasks or has a nearly full context window |
| 2. Workflows and tools | Commands and skills that run fixed steps, scripts, templates | Builds the right steps into how the work is done | Holds whenever the workflow is used |
| 3. Hooks, scripts and linters | Checks at the moment of action | Refuses or flags the action as it happens | Runs on every matching event |
| 4. Validation and tests | Test suites, merge gates, review | Catches what slipped through before it lands | Runs before every change lands |

Layers 2 to 4 do the enforcing. Layer 1 is the second line of defence: a context-file line can tell the agent about a guard, or cover knowledge that no check can encode, such as taste.

Research on coding agents points the same way. In one study, corrections kept in memory still left 57.5% of the checked preferences violated, while compiling them into runtime checks cut out-of-distribution violations to 2.0%. In another, the share of instructions followed fell steeply as the count rose from one to twenty. The figures and caveats are in [Evidence for enforcing over instructing]({{ '/sections/17-building-a-harness/self-improving-harnesses.html' | relative_url }}#evidence-for-enforcing-over-instructing).

A skill can sit in either layer. A skill whose text describes how to review code guides the model (layer 1). A skill that runs a script, fills a template or calls a fixed sequence of commands builds the steps in (layer 2).

<!-- IMAGE NEEDED: the four layers as a vertical stack around one agent action, such as an edit or a push, showing where each layer acts: instructions before the action, workflows shaping it, hooks at the moment it runs, tests before the result lands. -->

> Warning: adding a new, detailed rule to a context file every time an agent does something wrong is an anti-pattern. The rules pile up, the agent applies them less consistently as they grow, and the same failure returns in a long or busy session. Put the fix into a workflow, a tool, a hook or a test, and keep at most one context-file line that names it. The [complexity creep page]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }}#ad-hoc-rules-after-every-mistake) covers this in detail.

### 1.2 Applying it to each gap

1. Notice the gap.
   - A failure, a stall, or a correction you have given twice.
2. Record it.
   - One dated line in a friction log, with a count of how often it happened.
3. Find the layer that should have caught it.
   - Usually layer 2, 3 or 4. A missing line in a context file is rarely the whole answer.
4. Change that layer.
   - Build the step into the workflow, add a check, or add a test.
5. Measure.
   - Watch the next sessions for the same gap, and for correct work the change now blocks.
   - Keep the change if the gap closed and nothing else broke; revert it otherwise.

The [improvement loop page]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}) takes one gap through all five steps, with the hook it produced.

## 2. Start from friction

Friction is the input to every change. A copied harness brings its author's friction and habits with it, so build from your own.

| Kind of friction | Example | Usual layer for the fix |
|---|---|---|
| Failure | Said "done" with a failing test | 3 or 4: a hook or a gate |
| Stall | Waited for an answer it could find itself | 2: a tool or a workflow step |
| Repeated correction | Edited a generated file twice | 3: a hook that refuses the edit |
| Your own repetition | Retyped the same review prompt | 2: a skill |

### 2.1 Keep a friction log

Start with a list. Each time something goes wrong, write one line with the date, what happened, how often you have seen it, and what it cost. Do not fix anything yet. After a week, sort by count and cost. An illustrative log:

```text
date        what happened                                     seen   cost
2026-11-02  agent pushed straight to main                      2      high
2026-11-03  agent forgot to run the tests before saying done   4      medium
2026-11-04  I retyped the same review prompt again             6      low each, adds up
2026-11-05  agent edited a generated file instead of its source 1     medium
```

### 2.2 Find the part of the system that let it through

| Symptom | Part that let it through | Where the fix goes |
|---|---|---|
| The agent did not know a project rule | Missing knowledge | A check or workflow step if the rule can be checked; a [context file]({{ '/sections/04-context-files/' | relative_url }}) line if it cannot |
| The agent knew the rule and broke it anyway | Missing check | A [hook or a test]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) |
| The output had the right content in the wrong shape | Missing structure | A [template or stub]({{ '/sections/07-templates-and-stubs/' | relative_url }}) |
| You typed the same long prompt again | Missing tool | A [skill or command]({{ '/sections/05-skills-and-plugins/' | relative_url }}) |
| The agent did something you would have refused | Missing approval | A [human gate]({{ '/sections/14-human-gates/' | relative_url }}) |

#### Worked example

The second row of the log, "forgot to run the tests before saying done", happened four times. Figure 1 shows the same case:

1. Find the part.
   - The rule was already in `CLAUDE.md`, and the transcripts showed the agent had read it.
   - So the missing part was a check at layer 3.
2. Change the system.
   - Add a `Stop` hook that refuses to end the session while the test command has not passed since the last edit.
   - Shorten the `CLAUDE.md` line so it names the hook.
3. Measure.
   - Watch the next week of sessions for the same failure, and for any correct finish the hook blocks.

The full walk through this example is on [the improvement loop page]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}#a-full-worked-example).

## 3. Prefer a fix that a program enforces

A rule in prose competes with everything else in the context window, so the agent follows it less reliably as a session grows long and busy. Workflows, hooks, scripts, tests and permission rules run the same way every time.

![Sequence diagram of a never-force-push rule moving down the ladder. Step 1: the agent reads the instruction and force-pushes anyway. Step 2: the agent writes a PreToolUse hook; next session the hook refuses the force push with exit 2 and tells the agent to push a new branch, which it does. Step 3: a permission rule denies force pushes before the command runs.]({{ '/assets/images/17-building-a-harness/enforcement-ladder-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 2. One rule moving from an instruction to a hook to a permission rule. Diagram made with archify.*

### 3.1 The enforcement ladder

The ladder lists the mechanisms from softest to strictest. Put a fix on the softest rung that will hold it, and move it down when agents keep getting past it. A permission rule sits at the bottom even though it belongs to layer 3: the action cannot run at all, so nothing gets past it.

| Rung | Layer | Cost to write | How reliably it holds | Good for |
|---|---|---|---|---|
| Instruction in a context file | 1 | Minutes | Drifts in long sessions | Knowledge, conventions, taste |
| Skill or template | 2 | An hour | Holds when the agent loads it | Repeated procedures |
| Script or linter | 3 | An hour or two | Holds when it runs | Checks with a clear yes or no |
| Hook | 3 | An hour or two | Runs on every matching event | Rules agents keep breaking |
| Test or merge gate | 4 | Varies | Stops a bad result before it lands | Correctness of the output |
| Permission rule | 3 | Minutes | The action is unreachable | Destructive or outward actions |

### 3.2 Budget for the cost of each guard

Each guard has a cost:

- It can block correct work.
  - Count its false blocks; see [Measuring a guard's cost]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }}#measuring-a-guards-cost).
- It adds time to every matching tool call.
  - Set a `timeout` on each hook; 10 to 30 seconds covers most checks.
- Its refusal messages take context.
  - Keep each [refusal message]({{ '/sections/17-building-a-harness/refusal-messages.html' | relative_url }}) to two or three lines that name the next step.

> Note: the agent reads a refusal message as its next instruction. After a bare "Blocked", agents tend to retry the same action. After "Force push is blocked; push a new branch and open a pull request", the next try can succeed.

## 4. Let agents do the improvement work

In a mature harness, an agent files the bug, writes the fix, adds the check and opens the pull request, and you review the result. An illustrative exchange, shortened:

```text
> This is the second time a session said "done" with a failing test. Fix the harness so it cannot happen again.

  Reading the last two transcripts...
  Both sessions ran the tests, then edited a file and did not re-run them.
  CLAUDE.md already has the rule, so another line there will not help. I will add a Stop hook
  that refuses to finish while any edit is newer than the last passing test run.

  Wrote: the hook script, its settings entry, and a test that feeds the hook a stale stamp.
  Test: the hook exits 2 on a stale stamp and 0 on a fresh one. Both pass.
  Shortened the CLAUDE.md line to point at the hook.
  Opened a pull request with the diff and the test output for your review.
```

### 4.1 Make prompting cheaper for both of you

Agents write most of the prompts in a large harness, so a short trigger helps them as much as it helps you.

- A coordinator briefs a sub-agent.
  - A template for the brief keeps the original request and the done criteria in every one.
- A skill expands into a procedure.
  - One word such as `/review-changes` replaces a paragraph, and the steps run the same way each time.
- You type the same paragraph twice.
  - Turn it into a [skill]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

### 4.2 Keep a person on the decisions that matter

| Kind of decision | Who makes it | Example |
|---|---|---|
| Routine harness fix with a test | An agent, with your review of the pull request | A new hook for a rule agents broke |
| Outward action | You | A push to a shared branch, a message to someone |
| Destructive action | You | Deleting data, rewriting history |
| Architectural choice | You | Replacing a workflow, adding a new layer |

[Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}) covers how to draw that line.

### 4.3 What the research says

Studies of self-improving harnesses describe one outer loop:

1. Capture traces of real runs.
2. Mine one bounded weakness from them.
3. Propose a small edit.
4. Evaluate it against a protected task set, and keep it only if nothing regresses.
   - The evaluator stays outside what the loop may change.
5. Promote it through someone other than the proposer, usually a person, and keep a record so it can be rolled back.

One study found that automatic harness changes did no better than simply running the model more, so compare any automatic loop against that baseline. Details and sources: [Self-improving harness research]({{ '/sections/17-building-a-harness/self-improving-harnesses.html' | relative_url }}).

## 5. Complexity creep

Every fix adds a mechanism. Over months, a harness can become a patchwork of overlapping rules, hooks and scripts that nobody can hold in their head, where a change in one place breaks another. You balance keeping things simple against building more systems around the models.

| How it happens | What to do about it |
|---|---|
| A new context-file rule after each mistake | Put the fix into a workflow, hook or test; keep one line that names it |
| Two rules written for the same failure at different times | Merge them into one |
| A rule compensated for a weakness an older model had | Retire it and re-test (see [Trends and model changes]({{ '/sections/18-trends/' | relative_url }})) |
| A guard fires often on correct work | Narrow it, or move it to a later gate |
| Reusable parts and personal settings in one file | Split them, so each can be tested on its own |

> Warning: schedule removal the same way you schedule fixes, or the harness only grows. The [complexity creep page]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }}) gives a pruning procedure and a way to measure each mechanism.

## 6. A possible first week

![Sequence diagram of one session with a first-week harness. You type a review command; the agent loads CLAUDE.md and the review skill; each edit triggers a PostToolUse hook that runs the formatter and linter; when the agent runs git push, the approval rule asks you, you approve, and the push runs.]({{ '/assets/images/17-building-a-harness/first-week-session.svg' | relative_url }}){: style="width:100%"}

*Figure 3. One session using all four first-week files. Diagram made with archify.*

1. Write one `CLAUDE.md` (layer 1).
   - The five or six things you keep telling the agent that no check can encode.
2. Turn the prompt you type most into one skill (layer 1, moving to layer 2 as you script its steps).
3. Turn the rule an agent broke into one hook (layer 3).
4. Write one approval rule for what needs your say-so, such as `git push` (layer 3).

Each of the four is under a page long. The [first-week page]({{ '/sections/17-building-a-harness/first-week.html' | relative_url }}) has a complete, copyable version of each file, and what to watch for in the week after.

## Examples from my own harness

- Guards that started as written rules.
  - The rule about not printing secrets and the rule about running the tests before a merge both began as context-file lines that agents broke.
  - Each moved into a hook or a gate, and the written rule shrank to a line that names the guard.
- Agents run most of the improvement work.
  - When a session hits friction the harness should have prevented, it files an issue.
  - A later session fixes the harness and adds a regression test for the fix.

To start: keep a friction log for one week before you build anything.

## Pitfalls and lessons

<details markdown="1">
<summary>Each added rule and tool costs tokens and time on every run</summary>

A line in a context file is read at the start of every session. A hook runs on every matching tool call. Individually they are cheap; together they decide how much of the context window is left for the task. Remove the ones that no longer help.

</details>

<details markdown="1">
<summary>What works depends on the model</summary>

A rule that helped one model generation can hurt the next. Re-test your rules when you change models, and see [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}).

</details>

## Pages in this section

- [The improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}), with the sub-topics [Self-improving harness research]({{ '/sections/17-building-a-harness/self-improving-harnesses.html' | relative_url }}) and [Writing refusal messages]({{ '/sections/17-building-a-harness/refusal-messages.html' | relative_url }})
- [A first-week harness]({{ '/sections/17-building-a-harness/first-week.html' | relative_url }})
- [Complexity creep and pruning]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }})

## Sources

- Self-improving harness research (mine failures, propose small bounded edits, keep only changes that do not regress, keep the evaluator outside the thing being changed): sources listed on [Self-improving harness research]({{ '/sections/17-building-a-harness/self-improving-harnesses.html' | relative_url }}); not yet verified against the primary sources.
- Automatic harness changes doing no better than running the model more: <https://arxiv.org/abs/2607.12227>, verified 2026-11-09.
- Claude Code hooks, including the `Stop` event, `timeout` and exit code 2: <https://code.claude.com/docs/en/hooks.md>, not re-checked for this page.

{% include page_nav.html %}
