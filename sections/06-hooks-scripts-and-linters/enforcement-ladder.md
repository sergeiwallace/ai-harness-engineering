---
title: "The enforcement ladder"
parent: "Hooks, scripts and linters"
nav_order: 1
---

{% include page_nav.html %}

# The enforcement ladder

As of 2026-11-09.

A harness can put a rule in several places, from a sentence the model reads to a check that runs before every merge. The ladder ranks them by how consistently they hold. This page covers the four layers, the five concrete mechanisms inside them, how to place a rule, what to do when an agent breaks one, and what each mechanism costs. It is part of [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}), and it applies the [systems approach to harness engineering]({{ '/sections/02-ai-harnesses/' | relative_url }}#a-systems-approach-to-harness-engineering) from the AI harnesses overview.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The principle

A fix belongs in the system, enforced by code wherever a program can check it.

1. Instructions guide the model, and the model applies them unevenly.
   - It drops a rule when it handles several tasks at once.
   - It forgets a rule after switching to another task and back.
   - It loses track of a rule as the context window fills, and a compaction can remove the rule from the window.
   - A sub-agent never sees a rule its brief did not carry.
2. For these reasons, instructions in a context file are never sufficient on their own over many sessions.
3. The enforcing is done by the layers that run as code or as fixed steps: workflows and tools; hooks, scripts and linters; validation and tests.
4. The instruction stays as a second line of defence. It tells the model why, so the model plans around the rule and the checks refuse it less often.

> Adding another detailed rule to a context file each time an agent does something wrong is an anti-pattern. The file grows, each rule is followed as unevenly as the last, and the mistake comes back. The correction goes into a workflow, tool, hook, script, linter or test.

## The four layers

![A five-step staircase from least to most enforced: context rule, path rule, skill, hook or guard, test or CI gate. Each step up is more consistent and more costly.]({{ '/assets/images/06-hooks-scripts-and-linters/enforcement-ladder.svg' | relative_url }})
*The five mechanisms on this page, from least to most consistently enforced. Steps 1 and 2 are layer 1; step 3 is layer 2; step 4 is layer 3; step 5 is layer 4.*

| Layer | Mechanisms on this page | Enforces | When it acts |
|---|---|---|---|
| 1 Instructions in context files, skills and prompts | Context rule, path rule | No; the model reads it | Whenever the model remembers it |
| 2 Workflows and tools | Skill, a script that does the procedure, a tool that accepts only valid input | Yes, once the workflow runs | During the task |
| 3 Hooks, scripts and linters | Hook or guard, linter, pre-commit script | Yes | At the moment of the action |
| 4 Validation and tests | Test suite, CI gate, ratchet | Yes | Before the change lands |

Layers 2 to 4 do the enforcing. Layer 1 is the second line of defence.

## The five mechanisms

| Step | Mechanism | Layer | Checked | Fits | Weakness |
|---|---|---|---|---|---|
| 1 | Context rule | 1 | Never; the model reads it | Preferences and judgment calls no program can check | Least consistently followed |
| 2 | Path rule | 1 | Never; loads for matching files | The same, for some files only | Loads only when a file tool touches a match |
| 3 | Skill | 2 | When it runs | Procedures with fixed steps | Nothing forces it to run |
| 4 | Hook or guard | 3 | At the moment the agent acts | An action that must always or never happen | Can block legitimate work |
| 5 | Test or CI gate | 4 | On the result | A property of the output | Later and slower |

### Step 1: context rule

A line in `CLAUDE.md` or `AGENTS.md`, loaded on every turn. Section 4 covers [context files]({{ '/sections/04-context-files/' | relative_url }}).

- It can carry judgment that no program can check: "prefer small functions", "ask before adding a dependency".
- It fails under load, after compaction and in a sub-agent whose brief left it out.
- Claude Code's docs call `CLAUDE.md` "context, not enforced configuration".

```markdown
- Prefer small functions; split one that does more than one thing.
- Ask before adding a new dependency.
```

### Step 2: path rule

A rule in `.claude/rules/` that loads only when the agent works on matching files.

- It costs nothing in sessions that never touch those files.
- It loads on file tools, so it is absent when the agent touches the same files through a shell command.

```markdown
---
paths: ["migrations/**"]
---
- Never edit a migration that has been merged. Write a new one.
```

### Step 3: skill

A procedure that runs the same steps each time it is used. Section 5 covers [skills]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

- It builds the steps into the work, so the agent does not have to remember them.
- A step that a program can do exactly is a script the skill runs, such as `release_check.py`.
- It holds only when it runs. A must-hold step inside it also needs a hook or a test.

### Step 4: hook or guard

Code that runs at the moment the agent acts and can refuse the action.

- It holds in every session and after every compaction.
- A good refusal names the allowed form, so the agent's next call is right.
- [Hooks]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}) covers events and exit codes; [Writing a guard]({{ '/sections/06-hooks-scripts-and-linters/writing-a-guard.html' | relative_url }}) covers refusals agents can work with.

### Step 5: test or CI gate

A check on the result at commit, push or merge.

- It catches what got past the earlier steps, including work by people.
- CI is the only step that runs on a clean machine.
- [Git hooks, CI and linters]({{ '/sections/06-hooks-scripts-and-linters/git-hooks-ci-and-linters.html' | relative_url }}) covers these checks.

## Placing a rule

Put each rule at the highest layer that can check it, and keep a context line for the reason. Ask three questions in order:

1. Can a program decide whether the rule was followed?
   - No: it stays in layer 1, as a context rule or a path rule.
   - Yes: go on.
2. Is it a procedure with steps?
   - Build it into a workflow: a skill, a script or a tool (layer 2).
   - Put any must-hold step in a hook or a test as well.
3. Is it about an action or about a result?
   - An action ("never force push") goes in a hook (layer 3).
   - A result ("release notes list every merged change") goes in a test or CI gate (layer 4).

### Worked example: six rules placed

| Rule | Program can decide? | Action or result | Where it lives |
|---|---|---|---|
| Prefer small functions | No | | Context rule; a linter's complexity limit catches the extreme cases |
| Never edit a merged migration | Yes | Action on some files | Hook on edits under `migrations/`, plus a path rule for the reason |
| Cut a release the same way each time | Steps | | Skill with a `release_check.py` step |
| Never force push | Yes | Action | Hook |
| Never commit a secret | Yes | Action and result | Secret scan at commit, and again in CI |
| Release notes list every merged change | Yes | Result | CI check against the merge log |

## When an agent breaks a rule

A broken rule is a gap in the system. The fix goes into the layer that can enforce it.

![Sequence diagram of one rule over three weeks. The author writes the rule into the context file; a compaction drops it; the agent runs the suite without the lock and two runs collide. The author adds a hook; the hook refuses the unwrapped command with the wrapped form; the agent runs the wrapped form. Afterward the context file still explains why, and CI runs the full suite on a clean machine.]({{ '/assets/images/06-hooks-scripts-and-linters/rule-climbs-ladder-sequence.png' | relative_url }})
*The test-lock rule. It was in the context file and in the window when it broke; a hook now refuses the unwrapped command.*

### The steps

1. Find what happened and which rule it broke.
2. Ask whether the rule was already written down.
   - If it was, the instruction layer has already failed once. Rewording it or adding a second, more detailed line repeats the anti-pattern.
3. Choose the layer that can enforce it, using the three questions above.
4. Build the check, with a refusal or failure message that names the fix.
5. Add a test for the check, with one case for the command that broke it.
6. Keep or add the context line as the reason, in one sentence.

### Worked example: the test lock

| Step | What happened |
|---|---|
| Incident | An agent ran the end-to-end suite on a shared machine without the shared test lock, and two runs collided |
| Was it written down | Yes, in the context file |
| Layer | 3: the command is an action a program can check |
| Check | A `PreToolUse` hook refuses the suite command unless it is wrapped in the lock command, and prints the wrapped form |
| Test | Cases for the bare command (refused) and the wrapped command (allowed) |
| Context line | Kept: it says why the lock exists |

### Keep the context line

| | Context line only | Hook only | Both |
|---|---|---|---|
| Agent knows the rule in advance | Usually | No | Yes |
| Rule holds after compaction | No | Yes | Yes |
| Refusals per session | None, and breaks | Many | Few |

<details markdown="1">
<summary>Retiring a check</summary>

A guard that refuses safe work often, or that the harness has outgrown, can be narrowed or removed. Before removing one, look at what it refused in the last month. If it still refuses real mistakes, narrow its pattern; if it refuses nothing, remove it and its registration together.

</details>

## What each mechanism costs

| Step | Build | Per use | Maintenance |
|---|---|---|---|
| 1 Context rule | A sentence | Tokens on every turn | Rewording; the file grows with each rule |
| 2 Path rule | A file with a pattern | Tokens when it loads | Keeping the pattern right |
| 3 Skill | A procedure, often with scripts | Tokens when it runs | Keeping steps current |
| 4 Hook | A script and tests | Time on every matching call | Commands change; false refusals |
| 5 CI gate | A job | Minutes per push | Flaky checks; slow pipelines |

## Pitfalls

- Adding a more detailed context rule after each mistake, when the rule was already in the window.
- Dropping the context line when a hook takes over, so the agent is refused without knowing why.
- A gate so strict it cannot be passed, which leads people and agents to bypass it.

## Sources

- "context, not enforced configuration": <https://code.claude.com/docs/en/memory.md>, verified 2026-11-09.
- Path-scoped rules in `.claude/rules/`: <https://code.claude.com/docs/en/memory.md>, verified 2026-11-09.
- Moving a must-hold rule into a hook: <https://code.claude.com/docs/en/skills.md>, verified 2026-11-09.
- The test-lock example is from the author's own harness, described conceptually. The ladder diagram was drawn for this page; the sequence diagram was made with archify.

{% include page_nav.html %}
