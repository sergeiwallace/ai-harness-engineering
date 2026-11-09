---
title: "Harness context files"
parent: "AI harnesses"
nav_order: 1
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Harness context files

As of 2026-11-09. Part of [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}). The full section on this topic is [Context files]({{ '/sections/04-context-files/' | relative_url }}).

A context file is the standing brief a harness hands the model at the start of every session. It is the cheapest component to add, and a long or vague one is followed poorly. This page covers what a context file is for, what belongs in one, how to write rules a model follows, and how a context file relates to the rest of the harness. Three sub-topic pages go deeper: [layering]({{ '/sections/02-ai-harnesses/context-file-layering.html' | relative_url }}), [size budgets]({{ '/sections/02-ai-harnesses/context-file-budgets.html' | relative_url }}) and [compiling]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a context file is

### The file names

| Tool | File it reads | Where |
|---|---|---|
| Claude Code | `CLAUDE.md` (or `AGENTS.md` when no `CLAUDE.md` exists) | `~/.claude/CLAUDE.md`, `./CLAUDE.md` or `./.claude/CLAUDE.md`, `./CLAUDE.local.md`, subfolders, `.claude/rules/*.md` |
| Codex | `AGENTS.md` (an `AGENTS.override.md` wins over it) | `~/.codex/AGENTS.md`, then each folder from the repository root down to the working folder |
| Gemini CLI | `GEMINI.md` | user folder, project root |
| Cursor, Copilot and others | `AGENTS.md` or their own rule files | project root |

`AGENTS.md` began as a shared convention so one file could serve several tools.

- If a project has only `AGENTS.md`, Claude Code reads it.
- If it has both, Claude Code reads only the `CLAUDE.md` files.
- A common pattern is a `CLAUDE.md` whose first line is `@AGENTS.md`, so the content lives in one file and both tools read it.

### How it reaches the model

The harness reads the files when the session starts and places their text in the context, ahead of your first message.

1. Claude Code loads `CLAUDE.md` and `CLAUDE.local.md` from the working folder and every folder above it, and concatenates them from the top of the tree down. Files are joined; none replaces another.
2. Files in subfolders load later, the first time the agent reads or edits a file in that folder.
3. Codex walks from the repository root down to the working folder, takes at most one file per folder, and stops adding files at 32 KiB by default (`project_doc_max_bytes`). Files closer to the working folder come later and override earlier guidance.
4. After `/compact`, Claude Code re-reads the project-root `CLAUDE.md` from disk. Folder files and path rules come back the next time a matching file is touched.

![A sequence diagram of context loading over one session. At session start the harness reads the user file and the project file with its imports and loads both, with the system prompt, into the model's window, where they stay all session. During the task the model edits tests/test_api.py; the harness finds a path rule matching tests and its test rules reach the model fresh, absent until then. The model reads web/app.tsx; the harness finds web/CLAUDE.md and its frontend rules reach the model. At compaction the harness sends a summary and reloads the files: chat detail is lost and the files return.]({{ '/assets/images/02-ai-harnesses/context-loading.png' | relative_url }})

*Which context files reach the model, and when. The diagram simplifies compaction: the project-root `CLAUDE.md` is re-read from disk at once, while folder files and path rules come back the next time a matching file is touched. Our own diagram, made with archify.*

> Claude Code's documentation describes `CLAUDE.md` as "context, not enforced configuration", and says that to block an action regardless of what Claude decides, you use a `PreToolUse` hook. The model reads the file and usually follows it. Nothing checks that it did.
{: .important }

## What belongs in one

### The day-one test

Write what a capable new teammate would need on their first day and could not work out from the code in ten minutes.

| Belongs | Stays out |
|---|---|
| Build, test and lint commands | What the code already says (function names, file layout the agent can list) |
| Conventions that are not visible in the code | General advice the model already follows ("write clean code") |
| A short note of the rules your hooks and tests enforce, so the agent knows why a refusal comes | A new detailed rule for every mistake an agent makes |
| Pointers to longer procedures | The procedures themselves, in full |
| What needs your approval | Secrets, tokens, internal hostnames |

### Four kinds of line

Almost every useful line is one of four kinds.

| Kind | Example | What it saves |
|---|---|---|
| Fact | "The API server listens on port 8080 in development." | a search, and a wrong guess |
| Command | "Run `make test-fast` while iterating and `make test` before committing." | the agent will run it as written |
| Rule | "Never edit files under `generated/`; change the schema and run `make codegen`." | a known mistake, with what to do instead |
| Pointer | "Before changing a schema, read `docs/migrations.md`." | the detail stays out of the always-loaded file |

- Commands are the most useful kind, because the agent acts on them directly.
- A rule without the safe alternative sends the agent looking for a way around it.
- A rule that matters should also be enforced by a hook or test; the line explains why the refusal will come.

## Writing rules a model follows

### Specific rules

| Weak | Strong |
|---|---|
| Be careful with the database. | Never run a migration against the shared database. Use `make db-local`. |
| Keep commits clean. | One logical change per commit. Run `make lint` first. |
| Do not break things. | Run `make test` before every commit and fix every failure, including old ones. |

The strong versions name a command, a path or a check. A model can act on them, and you can tell whether it did.

### The reason in a clause

A rule with its reason survives cases the rule did not foresee.

- Without the reason: "Use `--ff-only` when updating the shared checkout."
- With it: "Use `--ff-only` when updating the shared checkout, because other sessions read it."
- What the reason buys: the agent sees that the same care applies to a shared branch it was never told about.

Keep the reason to one clause. A paragraph of justification costs tokens on every turn and is skimmed.

### Put the rule where the mistake happens

A rule loaded at the moment it applies is followed more reliably than one loaded hours earlier.

| Rule about | Better home | Loads |
|---|---|---|
| test style | a path rule on `tests/**` | when the agent opens a test |
| a frontend convention | `web/CLAUDE.md` | when the agent works in `web/` |
| pushing | a hook on `git push` with the rule in its refusal | at the push |
| a release step | the release skill | when the release starts |

See [Context file layering]({{ '/sections/02-ai-harnesses/context-file-layering.html' | relative_url }}#path-rules).

### Emphasis

Capital letters and "IMPORTANT" help a rule stand out until every rule uses them.

- One or two emphasised rules in a file are noticed.
- When most lines are emphasised, none stand out, and the file reads as noise.
- A rule that needs emphasis to be followed is a rule that should be enforced in code. Move it to a [hook]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}) or a test and keep a short plain line.

## A worked example

### Before

```markdown
# Notes
This is a Python project. Please write good code and tests. We use pytest.
Be careful with the database. Don't push without checking. The frontend
is in web/. Remember to update docs. Also use type hints.
```

### After

```markdown
# Project context

## Commands
- `uv run pytest -q` runs the tests. `uv run pytest -q -m "not slow"` while iterating.
- `uv run ruff check . && uv run mypy src` before every commit.

## Layout
- `src/` is the Python service. `web/` is the frontend; see `web/CLAUDE.md` for its rules.

## Rules
- Write the failing test first, then the fix.
- Never run migrations against the shared database. Use `make db-local`.
- Update the matching page in `docs/` in the same commit as the feature.

## Ask before
- Pushing to `main`, deleting files outside `src/` and `tests/`, adding a dependency.
```

<details markdown="1">
<summary>What changed and why</summary>

1. Every vague line became a command or a check ("good code and tests" became the test command and the test-first rule).
2. The frontend rules moved to a subfolder file that loads only when the agent works there.
3. "Don't push without checking" became an explicit approval list.
4. Lines the model follows anyway ("use type hints") were folded into the check that enforces them (`mypy`).

</details>

## Where a rule belongs

| If the rule is... | Put it in |
|---|---|
| Always true, short, for every session | the context file |
| True for some files only | a path rule |
| A procedure with steps | a [skill]({{ '/sections/02-ai-harnesses/skills.html' | relative_url }}) |
| Something that must hold every time | a [hook]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}) or a test |
| An action the agent may never take | a [permission deny rule]({{ '/sections/02-ai-harnesses/permissions-and-settings.html' | relative_url }}) |
| A lesson from one session | [memory]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}) |

## Maintenance

### Stale lines

A line that names a command that no longer exists costs more than naming nothing: the agent runs it, fails, and loses turns working out why.

> Update the context file in the same commit as the change that made a line stale. A rename of `make test` to `make check` that leaves the old name in `CLAUDE.md` breaks the first command every new session runs.
{: .warning }

A test can catch this: one that reads the context file, extracts every command in backticks that starts with `make`, and checks the target exists.

### Prune on a schedule

A rule nobody breaks any more may be compensating for an older model.

1. Every few months, read the file line by line.
2. For each line, ask whether removing it would change behaviour, and whether a hook or test already enforces it.
3. Remove what changes nothing, and shorten what a check already enforces to a pointer.
4. Keep a short note of what you removed, so you can restore a line if the mistake returns.

### When an agent makes a mistake

Adding another detailed rule to the context file each time an agent does something wrong is an anti-pattern. The file grows, each rule gets less attention, and the model still applies them unevenly when it juggles several things, switches tasks or works with a nearly full window. Following the [systems approach]({{ '/sections/02-ai-harnesses/' | relative_url }}#a-systems-approach-to-harness-engineering), the fix goes into the system first:

1. Ask which part of the system let the mistake through.
2. Build the fix into a workflow or tool where you can: a skill step, a script that does the fiddly part, a template with the right fields.
3. Where the action itself is the problem, add a hook, script or linter that refuses or flags it as it happens.
4. Add a test or validation step that catches the case before a change lands.
5. Only then, if it helps the model behave in line with the checks, add or adjust one short context-file line as a second layer.

In my harness, a request to record a rule goes through one procedure that first asks whether code can enforce it, then decides where any remaining text belongs (global file, one project's file, a path rule or a procedure document) and ships it.

## Commands and settings

| Command or setting | Does |
|---|---|
| `/init` | drafts a `CLAUDE.md` from the repository; edit it before you commit it |
| `/memory` | lists every context and memory file the session can load, and opens one for editing |
| `/context all` | shows the context files that actually loaded, under Memory files |
| `@path/to/file` | imports another file in place, up to four hops deep |
| `claudeMdExcludes` | a settings key that skips named `CLAUDE.md` files by path or glob, for large monorepos |
| `project_doc_max_bytes` | the Codex limit on how much `AGENTS.md` text it reads |
| `project_doc_fallback_filenames` | other file names Codex treats as instruction files |

## Pitfalls and edge cases

| Situation | What happens | What to do |
|---|---|---|
| two files contradict each other | Claude may pick one arbitrarily | keep each rule in one place; review nested files and rules together |
| `CLAUDE.md` and `AGENTS.md` both exist | Claude Code reads only `CLAUDE.md` | import `AGENTS.md` from `CLAUDE.md` |
| a path written in backticks, such as `` `@README` `` | it is not imported | intended: backticks keep a path literal |
| an import pointing outside the project | Claude Code asks once for approval before loading it | approve it, or move the file into the project |
| `CLAUDE.local.md` with several git worktrees | it exists only in the worktree where you made it | import a file from your home folder instead |
| a file over 4 MiB | Claude Code skips it entirely | split it; the 200-line guidance keeps you far from this |
| an instruction given only in chat | it is lost at compaction | put lasting rules in a file |

<!-- IMAGE NEEDED: a short CLAUDE.md file annotated with the four kinds of line (fact, command, rule, pointer); search: CLAUDE.md example annotated, AGENTS.md example file, good CLAUDE.md template -->

## Sub-topics

- [Context file layering]({{ '/sections/02-ai-harnesses/context-file-layering.html' | relative_url }}): scopes, precedence, imports, path rules and subfolder files.
- [Context file size budgets]({{ '/sections/02-ai-harnesses/context-file-budgets.html' | relative_url }}): what size costs, how to measure it, and the trim ladder.
- [Compiling context files]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}): building several tools' files from one set of source modules.

## Sources

- Claude Code memory and `CLAUDE.md`: <https://code.claude.com/docs/en/memory.md>, verified 2026-11-09.
- Codex `AGENTS.md` discovery: <https://learn.chatgpt.com/docs/agent-configuration/agents-md>, verified 2026-11-09.
- The `AGENTS.md` convention: <https://agents.md/>.
- The maintenance practices are from the author's own harness, described conceptually.

{% include page_nav.html %}
