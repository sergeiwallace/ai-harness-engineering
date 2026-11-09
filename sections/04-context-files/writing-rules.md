---
title: "Writing rules that get followed"
parent: "Context files"
nav_order: 2
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# Writing rules that get followed

A context file is read and interpreted by the model, so how a rule is written decides how often it is followed. This page covers what a context file can and cannot do, what to put in one, how to phrase each rule, what to do when a mistake repeats, and how to keep the file small and true.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a context file can do

A context file guides the model's general behaviour. It is applied unevenly, and it never enforces anything.

- It is followed most of the time, and less often when the model:
  - juggles several things at once
  - switches from one task to another
  - works with a nearly full context window.
- Claude Code's docs call `CLAUDE.md` "context, not enforced configuration".
- So in a harness it is the second layer. The layers, in order of how reliably they hold:

| Layer | What it does | Example: running only the changed tests |
|---|---|---|
| 1. Instructions: context files, skills, prompts | Guide behaviour and explain the checks | "`make test` runs changed tests; the full suite runs at push." |
| 2. Workflows and tools | Build the right step in | `make test` runs only the changed tests by default |
| 3. Hooks, scripts and linters | Encode how the output should be, and refuse or flag a mistake at the moment of action | A PreToolUse hook refuses `make test-all` outside a push |
| 4. Validation and tests | Catch what slips through before a change lands | The pre-push gate runs the full suite once |

Layers 2 to 4 do the enforcing. A context-file line is at most a second layer on top of them; the rest of this page is about writing that layer well.

Claude Code's own documentation draws the same line for organizations:

| To do this | Use |
|---|---|
| Block specific tools, commands or file paths | Settings: `permissions.deny` |
| Enforce sandbox isolation | Settings: `sandbox.enabled` |
| Code style and quality guidelines | A `CLAUDE.md` file |
| Behavioural instructions for the agent | A `CLAUDE.md` file |

For anything that must run at a fixed point, such as before every commit or after each edit, the docs point to a hook. [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers layer 3.

## What to write

### Belongs

- Commands the agent cannot guess: build, test, lint, run one test, start the app.
- Conventions that differ from the language default.
- Things never to do, with the reason and the right alternative.
- A short line for each check the system enforces, saying what it refuses and why, so the agent knows what to do when it is refused.
- Facts about the environment that mislead, such as an error message that points at the wrong cause.

### Does not belong

| Content | Where it goes instead |
|---|---|
| Long procedures | A spoke document, linked with a trigger ([Hub and spoke]({{ '/sections/04-context-files/hub-and-spoke-and-path-rules.html' | relative_url }})) |
| Facts the agent can read from the code, such as the layout | Nowhere; it reads them |
| Rules that must hold every time | The context file and a [hook or check]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) |
| Rules for one kind of file | A [path rule]({{ '/sections/04-context-files/hub-and-spoke-and-path-rules.html' | relative_url }}#path-scoped-rules) |

## How to phrase a rule

### The three parts

A rule the agent can act on names the action, the condition and the alternative.

| Part | Question | Example |
|---|---|---|
| Action | What exactly to do or not do | "Never run migrations" |
| Condition | When it applies | "against production" |
| Alternative | What to do instead | "Use `make migrate-staging`." |

### Weak and strong

| Weak | Strong |
|---|---|
| Be careful with the database. | Never run migrations against production. Use `make migrate-staging`. |
| Write good tests. | Every bug fix starts with a failing test that reproduces the bug. |
| Don't break the build. | Run `make check` before every commit; it must pass. |
| Keep it simple. | Add no new dependency without asking; prefer the standard library. |

### Give the reason

A rule with its reason is applied more sensibly to cases it did not name.

```markdown
- Money is integer cents, never floats: floats lose cents in sums.
```

Faced with a currency-conversion function the rule did not mention, the agent still keeps integer cents, because it knows why.

### Structure and consistency

- Group related rules under Markdown headings and bullets; organised sections are followed more reliably than dense paragraphs.
- Two rules that contradict each other get one picked arbitrarily. Check the root file, subdirectory files and `.claude/rules/` together.
- Check whether a rule competes with guidance the client adds on its own. If your file sets commit or pull request rules, Claude Code's `includeGitInstructions` setting turns its built-in ones off.

### Be specific about commands

```markdown
- Run one test: `pytest tests/path/test_file.py::test_name -q`.
- Run the full suite only before a push: `make test` (about 40 minutes).
```

The time in the second line tells the agent the cost of getting it wrong.

## Rules in the example file

The [overview]({{ '/sections/04-context-files/' | relative_url }}#a-complete-example) has a complete example file. Here is how four of its lines were built.

| Line | Action | Condition | Alternative or reason |
|---|---|---|---|
| "Full suite (`make test`, about 12 minutes): only before a push." | run the full suite | only before a push | the time says why |
| "Money is integer cents, never floats: floats lose cents in sums." | use integer cents | always | the reason |
| "Never run migrations against production. Use `make migrate-staging`." | never run migrations | against production | `make migrate-staging` |
| "`tests/integration/` needs `docker compose up db` first; the error says 'connection refused'." | start the database | before integration tests | the misleading error, so the agent recognises it |

### Known traps

The last block of the example is known traps. Each entry records something that looked like one problem and was another.

- Write the symptom the agent will see as well as the cause; the symptom is what it is looking at when it needs the line.
- Keep each trap to one or two lines.
- When a trap can be detected in code, fix it there instead, for example a setup script that starts the database, and keep the line only as the explanation.

## When a mistake repeats

### The anti-pattern

Adding another detailed rule to the context file each time the agent does something wrong does not hold over time.

- The file grows, and a long file is followed less reliably.
- Each new rule is applied unevenly, like every other line.
- The mistake comes back when the model is busy, switching tasks or near a full window.

### Fix it in the system

![Sequence: the agent runs the full 40-minute suite for one test twice in a week; you change make test to run changed tests by default, add a hook that refuses test-all outside a push, and add one line explaining the check; next week make test runs three tests in 8 seconds, and after a task switch with the window 85% full the agent tries test-all and the hook refuses it]({{ '/assets/images/04-context-files/correction-to-rule-sequence.svg' | relative_url }})

*A repeated mistake fixed in the system. The workflow change and the hook do the enforcing; the context-file line explains the check. In the last step the rule was missed and the hook held. Diagram made with archify; the timings are illustrative.*

1. Notice the pattern. A rule of thumb: if you have typed the same correction or the same prompt twice, consider building it into the harness.
   - Use judgement to fix it the first time when the context shows it will recur, or when a repeat would be costly, such as a deleted file or a push to the wrong branch.
   - A one-off correction in a one-off task can stay in the conversation.
2. Change the system at the lowest layer that can prevent it.
   - Workflow or tool: make the right path the default one.
   - Hook, script or linter: refuse the wrong action and print the right one.
   - Test or validation: catch it before the change lands.
3. Add a short context-file line that explains the check, so a refusal is not a surprise.
4. Watch whether the check fires. If it fires often, the workflow should make the right path easier.

### Worked example

The agent runs the full end-to-end suite, which takes forty minutes, when one test would do. It happens twice in a week.

The workflow change, in the `Makefile`:

```make
test:        ## changed tests only, the default
	files=$$(git diff --name-only origin/main -- 'tests/*.py'); \
	if [ -n "$$files" ]; then pytest $$files -q; else echo 'no changed tests'; fi
test-all:    ## the full suite; runs in the pre-push gate
	pytest tests -q
```

The hook, in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "jq -r '.tool_input.command' | grep -q 'make test-all' && { echo 'test-all runs at push; use make test' >&2; exit 2; } || exit 0" }]
      }
    ]
  }
}
```

The second-layer line, in `CLAUDE.md`:

```markdown
- `make test` runs the changed tests. The full suite (`make test-all`) runs at push; a hook refuses it otherwise.
```

| Layer | What changed | What it stops |
|---|---|---|
| Workflow | `make test` defaults to changed tests | The slow run happening by default |
| Hook | Refuses `make test-all` outside a push | The slow run happening when the rule is missed |
| Context file | One line explaining both | The agent being surprised by the refusal |

The files are illustrations written for this page; the hook's exit code 2 blocks the call and shows the message to the agent.

## Keeping the file small

### Why

- Everything in an always-loaded file costs context on every turn.
- A long file gets followed less reliably.
- Claude Code's docs suggest keeping each `CLAUDE.md` under 200 lines; Codex stops reading `AGENTS.md` files at 32 KiB in total.
- Moving a rule into a check (above) removes the detail from the file and leaves one explaining line.

### A pruning pass

| Ask of each line | If the answer is no |
|---|---|
| Did a recent session need it? | Delete it or move it to a spoke |
| Is it still true? | Fix it in the same commit as the change that broke it |
| Does a newer model still make this mistake? | Delete it |
| Is it a procedure? | Move it to a spoke with a one-line trigger |

## Keeping the file true

Context files go stale when the mechanisms they describe change.

1. A file that still names a retired script or an old test command sends every session the wrong way.
2. Treat the line as part of the mechanism it describes, and change both in one commit.
3. Where you can, let a check find stale lines, for example a test that every command named in `CLAUDE.md` exists in the `Makefile`.
4. Run `/doctor prompt-audit` in Claude Code. It reports instructions written for older models, references to files or commands that do not exist, and files that contradict each other, and changes nothing until you ask.
5. The `/doctor` checkup also proposes trims for a checked-in `CLAUDE.md`: it cuts what the agent can derive from the code (layouts, dependency lists, architecture overviews) and keeps pitfalls, reasons and conventions that differ from defaults.

<details markdown="1">
<summary>A review checklist for a context-file change</summary>

- Does each new line name an action, a condition and an alternative?
- Is there a reason where the rule would otherwise be misapplied?
- Did the change remove or fix any line the same commit made untrue?
- Is the file still inside its size budget?
- Is any must-hold rule also enforced in code?

</details>

## Sources

- The 200-line target, "context, not enforced configuration", writing effective instructions, the settings-versus-`CLAUDE.md` table, `includeGitInstructions` and `/doctor prompt-audit`: [Claude Code: how Claude remembers your project](https://code.claude.com/docs/en/memory.md), read 2026-11-09.
- The method and examples are from the author's own harness, described conceptually, and written for this page.

{% include page_nav.html %}
