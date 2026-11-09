---
title: "Context file layering"
parent: "Harness context files"
grand_parent: "AI harnesses"
nav_order: 1
---

{% include page_nav.html %}

# Context file layering

As of 2026-11-09. A sub-topic of [Harness context files]({{ '/sections/02-ai-harnesses/context-files.html' | relative_url }}), in [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}).

A session rarely reads one context file. It reads several, from broad to narrow, and combines them. Knowing which files load, when, and which wins when two disagree is what lets you put each rule exactly where it applies.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The layers

![Context files layer from broad to narrow: organisation policy, user file, project file, path rules, subdirectory files.]({{ '/assets/images/02-ai-harnesses/context-layering.svg' | relative_url }})

### Organisation policy

A managed file set by an administrator, read for every user on the machine. It holds rules the organisation requires, such as data-handling limits. Users cannot override it.

### User file

`~/.claude/CLAUDE.md` for Claude Code, `~/.codex/AGENTS.md` for Codex. It applies to every project you open: your writing style, your safety rules, how you like to be asked questions. It is never committed to a project, so it is the place for preferences your teammates do not share.

### Project file

`CLAUDE.md` or `AGENTS.md` at the repository root, committed and shared. It holds the build and test commands and the project's conventions. A `CLAUDE.local.md` next to it can hold your own project-specific notes and stays out of git.

### Path rules

Files in `.claude/rules/` with a `paths` pattern in their front matter. They load only when the agent reads or edits a matching file.

### Subfolder files

A `CLAUDE.md` inside a folder loads when the agent starts working in that folder. A monorepo uses these to give each package its own rules.

## Loading and precedence

### When each layer loads

| Layer | Loads | Survives compaction |
|---|---|---|
| Organisation, user, project | at session start | yes, reloaded |
| Path rule | when a matching file is touched | reloaded on the next match |
| Subfolder file | when the agent works in the folder | reloaded on the next visit |

### Which wins

The files are combined, not replaced. When two say different things, the narrower and later one usually wins in practice, because it is closer to the task in the context and more specific. Organisation policy is the exception: it is meant to hold whatever a narrower file says. Avoid relying on precedence at all. If two files disagree, fix one of them.

> A rule that appears in two layers costs tokens twice and drifts apart over time. Put each rule in exactly one layer.
{: .warning }

## Imports

### The import line

A Claude Code context file can pull in another file with an `@` line:

```markdown
# Project context

@AGENTS.md
@docs/conventions/testing.md
```

The imported text loads as if it were written in place. This is how one `AGENTS.md` serves both Codex and Claude Code: the `CLAUDE.md` is a single import line.

### Imports and cost

An imported file loads every time its parent does. Importing a long procedure into the user file puts that procedure in every session. A pointer ("Before a release, read `docs/release.md`") costs one line and loads the procedure only when needed.

## Path rules

### Anatomy

```markdown
---
paths:
  - "tests/**/*.py"
  - "**/test_*.py"
---

# Test rules

- Test behaviour at the real entry point: the command, the endpoint, the page.
- One assertion per behaviour; name the test after the behaviour.
- Never mock the module under test.
```

### When to use one

A rule that applies only to some files is the best candidate. The rule is absent while the agent edits documentation and present the moment it opens a test, which is when it matters. My harness keeps its test standards, its writing rules for public documents and its skill-authoring rules as path rules, so each loads only for the files it governs.

### A worked example

![A sequence diagram of context loading over one session. At session start the harness reads the user file and the project file with its imports and loads both, with the system prompt, into the model's window, where they stay all session. During the task the model edits tests/test_api.py; the harness finds a path rule matching tests and its test rules reach the model fresh, absent until then. The model reads web/app.tsx; the harness finds web/CLAUDE.md and its frontend rules reach the model. At compaction the harness sends a summary and reloads the files: chat detail is lost and the files return.]({{ '/assets/images/02-ai-harnesses/context-loading.png' | relative_url }})

*Which context files reach the model, and when. Our own diagram, made with archify.*

An agent spends an hour on documentation, then switches to tests. A test-style rule written in the project file has been in the context all that hour, buried under an hour of messages. The same rule as a path rule arrives fresh at the moment the agent opens the first test file.

## Layering in a multi-machine harness

My harness runs on several machines with different tools and access. The global context file has a shared base, and each machine adds an overlay section that narrows or overrides the base for that machine: which tools are installed, which actions need approval there, which credentials route where. The overlay is selected by an environment variable set at install time, never by the machine's hostname, so a renamed machine does not pick up the wrong rules.

<details markdown="1">
<summary>Why an overlay and not a separate file</summary>

A separate file per machine repeats the base, and the copies drift. An overlay is a short section that says only what differs, and the build places it after the base so it reads as the narrower layer. See [Compiling context files]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}).

</details>

## Related pages

- [Context file size budgets]({{ '/sections/02-ai-harnesses/context-file-budgets.html' | relative_url }}): every layer adds to the always-loaded total.
- [Compiling context files]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}): building the layers from source modules.
- [Context files]({{ '/sections/04-context-files/' | relative_url }}): the full section.

## Sources

- Claude Code memory, imports and rules: <https://code.claude.com/docs/en/memory.md>, verified 2026-11-08.
- Codex `AGENTS.md` discovery: <https://developers.openai.com/codex/guides/agents-md>.
- The machine overlay is from the author's own harness, described conceptually.

{% include page_nav.html %}
