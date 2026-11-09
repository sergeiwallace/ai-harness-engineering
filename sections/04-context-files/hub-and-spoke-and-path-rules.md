---
title: "Hub and spoke and path rules"
parent: "Context files"
nav_order: 3
redirect_from:
  - /sections/04-context-files/advanced.html
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# Hub and spoke and path rules

Two techniques keep the always-loaded part of the context small without losing the detail:

- Hub and spoke keeps one line per rule in the context file and moves the detail to procedure documents, or skills, that load when a task needs them.
- Path-scoped rules keep the rules for one kind of file out of the window until the agent works on such a file.

This page covers both, how they fail, and when to use which.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Hub and spoke

### The pattern

```text
CLAUDE.md (hub, always loaded)              docs/ (spokes, read on a trigger)
- Before a release, read docs/release.md.   ──► release.md     12 steps
- Before a migration, read docs/migrations.md ► migrations.md  checks, rollback
- When writing tests, read docs/testing.md. ──► testing.md     conventions, fixtures
```

1. The context file (the hub) holds one line per rule, with a trigger and a link to a procedure document (a spoke).
2. The hub loads every session; the spokes do not.
3. When a task matches a trigger, the agent reads that spoke.
   - The spoke's steps enter the window for that task only.
   - The other spokes stay unread.
4. The agent follows the spoke's steps and reports back.

### In a task

![Sequence during a release task: the hub line is loaded, the trigger matches, the agent reads the spoke, follows its steps and reports back]({{ '/assets/images/04-context-files/hub-and-spoke-sequence.svg' | relative_url }})

*The hub costs one line on every turn; the spoke costs its full length only during a release. Diagram made with archify.*

### Writing the hub line

#### Put the trigger in the line

The agent has to know when to follow a link. A bare list of links gets ignored.

| Ignored | Followed |
|---|---|
| `See also: docs/release.md` | `Before any release, read docs/release.md.` |
| `Docs: testing, migrations, review` | `Before writing a migration, read docs/migrations.md.` |

#### Keep the rule itself in the hub

Put the rule's core in the line, so it applies even if the agent does not open the spoke:

```markdown
- Releases: tag from main only, after the full suite passes. Steps: before any release, read docs/release.md.
```

### Writing the spoke

A spoke is a procedure: numbered steps, the commands, the checks, what to do when a step fails. It can be long, because it loads only when needed.

```markdown
# Release procedure

1. Confirm you are on main and up to date: `git switch main && git pull --ff-only`.
2. Run the full suite: `make test`. Stop if anything fails.
3. Update CHANGELOG.md from the commits since the last tag.
4. Tag: `git tag v<version> && git push origin v<version>`.
5. If the publish job fails, do not retag; open an issue with the job log.
```

### Trade-offs

| Gain | Cost |
|---|---|
| Small always-loaded file | One extra read when a task needs the detail |
| Detail can be long and precise | The agent may skip the read if the trigger is vague |
| Each procedure has one home | Linking out moves the size problem into the spokes |

### Spoke document or skill

Claude Code's documentation recommends skills for task-specific instructions that do not need to be in context all the time. A skill is a spoke with its trigger built in.

| | Spoke document | Skill |
|---|---|---|
| What starts the read | The agent follows the trigger in the hub line | The model matches the skill's description to the task, or you type `/name` |
| Always in context | The hub line | The skill's name and description |
| Loads in full | When the agent reads the document | When the skill is invoked |
| Works in both clients | Yes, any Markdown file | Both have skills, in different folders |
| Good for | Reference material several tasks point to | A repeatable procedure with its own steps and checks |

In practice a release procedure fits well as a skill, and a long style guide that several tasks consult fits well as a spoke. See [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

## Path-scoped rules

### How they work

A path rule is a Markdown file in `.claude/rules/` with a `paths` field of globs in its YAML frontmatter.

1. At launch, rules without `paths` load like a context file; rules with `paths` wait.
2. When the agent reads, writes or edits a matching file, the rule loads. Viewing one file with `cat` or `head` counts as a read.
3. It stays loaded until a compaction summarises it away, and loads again on the next matching file.

```markdown
---
paths: ["docs/**/*.md"]
---
- Every new page gets a "Sources" section with a verification date per source.
- No em dashes; use a comma or a full stop.
```

### Writing the paths

| Detail | Rule |
|---|---|
| Format | A YAML list, or a comma-separated string |
| Base directory | Project rules: the directory that contains the rule's `.claude` folder. User rules in `~/.claude/rules/`: the session's working directory |
| `**/*.ts` | `.ts` files anywhere under the base directory |
| `src/**/*` | Every file under `src/` |
| `/*.md` | `.md` files directly in the base directory only |
| Braces | `src/**/*.{ts,tsx}` matches both extensions |
| Other frontmatter fields | Ignored; `paths` is the only field read |

### Sharing rules

- `~/.claude/rules/` holds personal rules that apply in every project on the machine.
- `.claude/rules/` supports symlinks, so one set of rules can be linked into several projects. A link whose target is outside the working directory is treated like an external import and needs approval once per project.

### When to use them

Use a path rule when the rule applies to one kind of file: tests, docs, migrations, infrastructure. It keeps those rules out of the window during unrelated work.

| Rule file | `paths:` | Loads when the agent touches |
|---|---|---|
| `tests.md` | `["**/tests/**", "**/*_test.go"]` | Any test file |
| `migrations.md` | `["**/migrations/**"]` | A migration |
| `infra.md` | `["infra/**", "**/*.tf"]` | Infrastructure code |
| `docs.md` | `["docs/**/*.md"]` | A documentation page |

### Limits

| Limit | What happens | What to do |
|---|---|---|
| Shell-only work | A file written with `cat > file` or a generator never loads the rule | Enforce the important rules with a hook or linter that checks the file |
| Compaction | A loaded rule is summarised away and returns only on the next matching touch | See [Context files after compaction]({{ '/sections/04-context-files/after-compaction.html' | relative_url }}) |
| Size | A rule that loads on demand and is over 40,000 characters (200K window) loads only its start | Split it into several rule files; each counts on its own |
| Codex | Path-scoped rules are not documented | Put an `AGENTS.md` in the folder that holds those files |

## Choosing between them

| The detail is | Use |
|---|---|
| A repeatable procedure for an occasional task (release, migration, incident) | A skill, or a spoke document with a trigger line |
| Reference material several tasks consult | A spoke document |
| Conventions for one kind of file, needed whenever such a file is edited | A path rule |
| Rules for one folder | A subdirectory `CLAUDE.md` |
| A rule that must hold every time | A hook, linter or test that enforces it, plus one of the above to explain it |

## Examples from my own harness

- Each procedure document is named by one line in the shared context file, with the trigger for reading it.
- Path-scoped rules cover tests, documentation and skill authoring, so those rules appear only when an agent works on those files.
- The parts that must hold are enforced elsewhere:
  - documentation pages go through a check for forbidden names and punctuation;
  - tests go through the test gate before a push;
  - the rules explain what those checks refuse.

## Sources

- Path rules (loading, glob syntax, base directories, frontmatter, symlinks, user-level rules, size limits) and the recommendation to use skills for task-specific instructions: [Claude Code: how Claude remembers your project](https://code.claude.com/docs/en/memory.md), read 2026-11-09; after compaction: [Claude Code: the context window](https://code.claude.com/docs/en/context-window.md), verified 2026-11-09.
- Hub and spoke is from the author's own harness, described conceptually. Example files are illustrations written for this page.

{% include page_nav.html %}
