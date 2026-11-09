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

A managed file set by an administrator, deployed with MDM, Group Policy or a similar tool.

| Platform | Claude Code path |
|---|---|
| macOS | `/Library/Application Support/ClaudeCode/CLAUDE.md` |
| Linux and WSL | `/etc/claude-code/CLAUDE.md` |
| Windows | `C:\Program Files\ClaudeCode\CLAUDE.md` |

- It is read for every user on the machine, in every project, and loads before user and project files.
- A developer cannot exclude it with `claudeMdExcludes`.
- Its content can also live in a `claudeMd` key inside managed settings.
- Claude Code's docs split the job: managed settings do the technical enforcement (`permissions.deny`, `sandbox.enabled`), and the managed `CLAUDE.md` carries behavioural guidance. That is the [systems approach]({{ '/sections/02-ai-harnesses/' | relative_url }}#a-systems-approach-to-harness-engineering) at organisation scale.

### User file

`~/.claude/CLAUDE.md` for Claude Code, `~/.codex/AGENTS.md` (or `AGENTS.override.md`) for Codex.

- Applies to every project you open.
- Claude Code also reads personal rules from `~/.claude/rules/`, which apply to every project on the machine.
- Holds your writing style, your safety rules, how you like to be asked questions.
- Never committed to a project, so it is the place for preferences your teammates do not share.

### Project file

`./CLAUDE.md` or `./.claude/CLAUDE.md` for Claude Code, `AGENTS.md` for Codex, at the repository root, committed and shared.

- Holds the build and test commands and the project's conventions.
- Reviewed like code, because it changes how every teammate's agent behaves.
- A `CLAUDE.local.md` beside it holds your own notes for the project. Add it to `.gitignore` yourself; nothing ignores it for you.
- Files in folders above the working folder load too, so starting in `packages/api/` still loads the repository's root file.

### Path rules

Markdown files in `.claude/rules/`, found recursively, one topic per file.

- A rule with a `paths` pattern in its front matter loads only when the agent reads or edits a matching file, and costs nothing while the agent works elsewhere.
- A rule without `paths` loads at launch, with the same priority as `.claude/CLAUDE.md`.
- The folder can hold symlinks to a shared set of rules; links that point outside the project need a one-time approval.
- Covered in depth under [Anatomy](#anatomy) below.

### Subfolder files

A `CLAUDE.md` inside a subfolder loads the first time the agent reads, writes or edits a file in that folder.

```text
repo/
  CLAUDE.md            project-wide commands and conventions
  web/CLAUDE.md        frontend rules, loaded in web/
  services/api/CLAUDE.md   API rules, loaded in services/api/
```

A monorepo uses these to give each package its own rules without loading all of them everywhere.

## Loading and precedence

### When each layer loads

| Layer | Loads | After `/compact` |
|---|---|---|
| Organisation, user, project root | at session start | the project-root file is re-read from disk |
| Rule without `paths` | at session start | loaded with the project files |
| Path rule | when a matching file is touched | loads again on the next match |
| Subfolder file | when a file in the folder is touched | loads again on the next touch |

To see what actually loaded, run `/context all` and read the Memory files list, or `/memory` for every location the session knows about.

### Which wins

The files are combined; each one adds to the context.

- Claude Code concatenates them from the top of the folder tree down to the working folder. Its docs say that when two instructions contradict each other, Claude may pick one arbitrarily.
- Codex orders them the same way and states that files closer to the working folder override earlier guidance.
- Neither tool resolves a contradiction reliably. If two files disagree, fix one of them.

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

| Rule | Detail |
|---|---|
| Relative paths | resolve from the folder of the file that contains the import |
| Depth | imports can import, up to four hops |
| Backticks | `` `@README` `` in backticks is literal text and is not imported |
| Spaces in a path | escape each space with a backslash |
| Outside the project | Claude Code asks once for approval before loading it |

### Imports and cost

An imported file loads every time its parent does.

| Approach | Cost | When the detail arrives |
|---|---|---|
| `@docs/release.md` in the user file | the whole procedure, every session | always |
| "Before a release, read `docs/release.md`." | one line | when a release starts |
| a release skill | one description line | when the skill runs |

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

A rule that applies only to some files is the best candidate.

| Rule | Path pattern |
|---|---|
| test standards | `tests/**`, `**/test_*.py` |
| public writing rules | `docs/public/**` |
| skill-authoring standards | `**/SKILL.md` |
| migration rules | `migrations/**` |

My harness keeps its test standards, its public writing rules and its skill-authoring rules as path rules, so each loads only for the files it governs.

### A worked example

![A sequence diagram of context loading over one session. At session start the harness reads the user file and the project file with its imports and loads both, with the system prompt, into the model's window, where they stay all session. During the task the model edits tests/test_api.py; the harness finds a path rule matching tests and its test rules reach the model fresh, absent until then. The model reads web/app.tsx; the harness finds web/CLAUDE.md and its frontend rules reach the model. At compaction the harness sends a summary and reloads the files: chat detail is lost and the files return.]({{ '/assets/images/02-ai-harnesses/context-loading.png' | relative_url }})

*Which context files reach the model, and when. The diagram simplifies compaction: the project-root `CLAUDE.md` is re-read from disk at once, while folder files and path rules come back the next time a matching file is touched. Our own diagram, made with archify.*

An agent spends an hour on documentation, then switches to tests. A test-style rule written in the project file has been in the context all that hour, buried under an hour of messages. The same rule as a path rule arrives fresh at the moment the agent opens the first test file.

## Layering in a multi-machine harness

My harness runs on several machines with different tools and access.

1. The global context file has a shared base.
2. Each machine adds an overlay section that narrows or overrides the base for that machine: which tools are installed, which actions need approval there, which credentials route where.
3. The build places the overlay after the base, so it reads as the narrower layer.
4. The overlay is selected by an environment variable set at install time, never by the machine's hostname, so a renamed machine does not pick up the wrong rules.

<details markdown="1">
<summary>Why an overlay and not a separate file</summary>

A separate file per machine repeats the base, and the copies drift. An overlay is a short section that says only what differs, and the build places it after the base so it reads as the narrower layer. See [Compiling context files]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}).

</details>

## Pitfalls

| Pitfall | Result | Fix |
|---|---|---|
| the same rule in the user file and the project file | paid for twice; the copies drift | keep it in one layer |
| a long procedure imported into the user file | loads in every session of every project | a pointer line or a skill |
| a monorepo root file with rules for one package | every package pays for them | a subfolder file or a path rule |
| a path rule whose pattern never matches | the rule never loads, silently | open a matching file and check `/context all` |

## Related pages

- [Context file size budgets]({{ '/sections/02-ai-harnesses/context-file-budgets.html' | relative_url }}): every layer adds to the always-loaded total.
- [Compiling context files]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}): building the layers from source modules.
- [Context files]({{ '/sections/04-context-files/' | relative_url }}): the full section.

## Sources

- Claude Code memory, imports and rules: <https://code.claude.com/docs/en/memory.md>, verified 2026-11-09.
- Codex `AGENTS.md` discovery: <https://learn.chatgpt.com/docs/agent-configuration/agents-md>, verified 2026-11-09.
- The machine overlay is from the author's own harness, described conceptually.

{% include page_nav.html %}
