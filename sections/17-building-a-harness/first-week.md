---
title: "A first-week harness"
parent: "Building your own harness"
nav_order: 2
last_updated: "2026-11-09"
talk_stop: "12"
stop_minutes: "5"
---

{% include page_nav.html %}

# A first-week harness

You can start a useful harness in a week with four small files. Each one covers a different kind of [friction]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}): something the agent did not know, something you keep typing, something the agent keeps doing wrong, and something you want to approve first. This page gives a complete version of each for Claude Code, with notes on the Codex equivalent.

![Sequence diagram of one session with a first-week harness. You type a review command; the agent loads CLAUDE.md and the review skill; each edit triggers a PostToolUse hook that runs the formatter and linter; when the agent runs git push, the approval rule asks you, you approve, and the push runs.]({{ '/assets/images/17-building-a-harness/first-week-session.svg' | relative_url }}){: style="width:100%"}

*Figure 1. How the four files act during one session. Diagram made with archify.*

| Day | File | Friction it removes | Deeper reading |
|---|---|---|---|
| 1 | `CLAUDE.md` | The agent did not know a project rule | [Context files]({{ '/sections/04-context-files/' | relative_url }}) |
| 2 | One skill | You retype the same long prompt | [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}) |
| 3 | One hook | The agent breaks a rule it knows | [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) |
| 4 | One approval rule | The agent does something you would have refused | [Human gates]({{ '/sections/14-human-gates/' | relative_url }}) |
| 5 | Nothing new | Read the week's friction log | [The improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}) |

## Day 1: one context file

### What goes in

The context file is layer 1, the guidance layer. Days 2 to 4 add the layers that enforce. Use this test for each candidate line:

| Candidate line | Keep it in `CLAUDE.md`? | Why |
|---|---|---|
| The command that runs the tests | Yes | The agent needs it to work at all |
| "Push branches, never main" | As a one-line pointer | Day 4's approval rule asks before every push, so you see a push to main before it happens |
| "Format code before committing" | As a one-line pointer | Day 3's hook enforces it |
| A point of taste, such as how to name functions | Yes | No check can encode it |
| The directory layout | No | The agent can read it for itself |

Any line you find yourself rewriting more emphatically after a mistake belongs on a later day, as a hook, a workflow or a test.

### A complete example

```markdown
# Project notes for agents

## Commands
- Run tests: `make test`. Run them after your last edit, not only before it.
- Format: `make fmt`. A hook runs it after each edit, so you do not need to.

## Rules
- Work on a branch and open a pull request. Every `git push` asks for approval.
- Do not edit files under `generated/`; edit the schema in `schema/` and run `make gen`.
- Ask before adding a new dependency.

## Taste
- Prefer a small function with a clear name over a comment that explains a large one.
- Error messages say what to do next, not only what went wrong.
```

#### Why it is short

Every line is read at the start of every session and competes with the task for attention. Keep it to the six to ten lines you have needed so far. The same file works for Codex if you name it `AGENTS.md`, or keep one and point the other at it.

## Day 2: one skill

### Pick the prompt you type most

A skill sits in layer 1 or layer 2, depending on what it does. A skill whose text tells the agent how to work is guidance (layer 1). A skill that runs a script, fills a template or calls fixed commands builds the steps into the workflow (layer 2). A first skill is usually the first kind, and moves toward the second as you script its steps.

1. Look back over the last week of sessions.
2. Find the instruction you typed most often.
   - For many people it is a review request, a commit message format or a status note.
3. Write its steps as a numbered list, in the order you want them run.

### A complete example

The skill is a folder named `review-changes` inside `.claude/skills/`, holding one `SKILL.md`:

```markdown
---
name: review-changes
description: Review the uncommitted changes in this repository for bugs, missing tests and unclear names. Use when asked to review changes or before opening a pull request.
---

# Review changes

1. Run `git diff` and read every changed file in full, not only the hunks.
2. For each change, check:
   - Does a test cover the new behaviour? If not, say which test is missing.
   - Can any input make it fail? Name the input.
   - Is every new name clear to someone who has not seen this diff?
3. Report findings as a numbered list, most serious first, each with the file and line.
4. Fix only what is clearly a bug. List the rest for the person to decide.
```

#### How it is used

1. You type `/review-changes`, or you ask for a review in your own words.
   - In the second case the agent loads the skill because its description matches the request.
2. The agent follows the numbered steps in `SKILL.md`.
3. It reports findings in the format the skill asks for.

The same `SKILL.md` format works in Codex, because Agent Skills is a shared format.

#### Moving it toward layer 2

This skill is mostly guidance: step 2 relies on the agent to notice a missing test. To enforce that step, give the skill a script and make step 2 run it.

```bash
#!/usr/bin/env bash
# list-untested.sh: print changed source files that have no matching test file.
for f in $(git diff --name-only -- 'src/*.py'); do
  t="tests/test_$(basename "$f")"
  [[ -f "$t" ]] || echo "no test file for $f (expected $t)"
done
```

Step 2 then begins "Run `tools/list-untested.sh` and report every line it prints", and the check no longer depends on the agent noticing.

## Day 3: one hook

### Pick the rule the agent broke

A hook is layer 3: it acts at the moment the agent edits a file or runs a command, whether or not the agent remembers the rule.

| Candidate | Hook event | Why it suits a first hook |
|---|---|---|
| Formatting and lint | PostToolUse on Edit and Write | Easy to check, costly to remind about |
| Edits to generated files | PreToolUse on Edit and Write | A clear yes or no on the file path |
| Finishing with untested edits | Stop | Catches the "done" claim itself |

The example below takes formatting.

### A complete example

The settings entry, in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "$CLAUDE_PROJECT_DIR/tools/format-edited-file.sh", "timeout": 30 }
        ]
      }
    ]
  }
}
```

The script it runs:

```bash
#!/usr/bin/env bash
# PostToolUse hook: format the file the agent just edited, and report lint problems back to it.
set -euo pipefail
file=$(jq -r '.tool_input.file_path // empty')
[[ -z "$file" || ! -f "$file" ]] && exit 0
case "$file" in
  *.py) ruff format --quiet "$file"; problems=$(ruff check --quiet "$file" || true) ;;
  *.ts|*.js) npx --no-install prettier --write --log-level warn "$file"; problems="" ;;
  *) exit 0 ;;
esac
if [[ -n "$problems" ]]; then
  echo "Lint problems in $file; fix them before moving on:" >&2
  echo "$problems" >&2
  exit 2
fi
```

#### What the exit codes mean

| Exit code | Effect in Claude Code |
|---|---|
| 0 | Success; the agent carries on |
| 2 | The message on standard error goes back to the agent, which then acts on it |
| Other | A non-blocking error, shown to you, not to the agent |

<details markdown="1">
<summary>Why a PostToolUse hook and not a PreToolUse one</summary>

A PreToolUse hook runs before the tool and can stop it, which is right for a dangerous command such as a force push. A PostToolUse hook runs after the edit has happened, so it cannot undo it, but it can fix the result and tell the agent what is still wrong. Formatting changes the file and does not need to stop anything, so it fits after the edit. The [enforcement ladder]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}#4-change-the-system) shows a PreToolUse example.

</details>

## Day 4: one approval rule

### Pick what needs your say-so

A permission rule is also layer 3, and the strictest rung: the action cannot run without you. Candidates:

- `git push`, so you see every change that leaves your machine.
- Deploy commands.
- Commands that send a message or an email.
- Deletions outside the working tree.

Start with one.

### A complete example

In the same `.claude/settings.json`:

```json
{
  "permissions": {
    "ask": ["Bash(git push:*)"],
    "deny": ["Bash(git push --force:*)", "Bash(git push -f:*)", "Read(./.env)"]
  }
}
```

| Key | Effect |
|---|---|
| `ask` | Claude Code stops and asks you before any `git push` |
| `deny` | The command is refused outright, and the agent cannot read your `.env` file |

A `Bash(...:*)` rule matches commands that start with that text, so each spelling needs its own entry: `git push --force` and `git push -f` are listed separately above. A force push written another way, such as `git push origin +main`, needs a hook that parses the command. Codex has its own approval and sandbox settings that play the same role.

## Day 5: read the friction log

Do not add a fifth file. Read the week's [friction log]({{ '/sections/17-building-a-harness/' | relative_url }}#21-keep-a-friction-log) instead, and ask two questions:

1. Which rule does the agent still break? That is next week's hook.
2. Which prompt do you still retype? That is next week's skill.

Then take the top item through the [improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}).

## Examples from my own harness

- It started the same way.
  - One context file with rules I kept repeating.
  - A few skills for prompts I typed every day.
- Most of what came later grew out of the friction log, one item at a time.
  - Many early written rules became hooks, workflows or tests.
  - Their context-file lines are now short pointers to the guard that enforces them.

To start: copy the four examples above, change the commands to your project's, and run one real task with them.

## Sources

- Claude Code hooks, events, matchers and exit codes: [Claude Code docs: Hooks](https://code.claude.com/docs/en/hooks.md), not re-checked for this page.
- Claude Code settings and permission rules: [Claude Code docs: Settings](https://code.claude.com/docs/en/settings.md), not re-checked for this page.
- Agent Skills format: [Claude Code docs: Skills](https://code.claude.com/docs/en/skills.md), not re-checked for this page.

{% include page_nav.html %}
