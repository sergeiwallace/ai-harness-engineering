---
title: "Hub and spoke and path rules"
parent: "Context files"
nav_order: 3
redirect_from:
  - /sections/04-context-files/advanced.html
---

{% include page_nav.html %}

# Hub and spoke and path rules

As of 2026-11-09. Part of [Context files]({{ '/sections/04-context-files/' | relative_url }}). Slide: [stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

Two techniques keep the always-loaded part of the context small without losing the detail. Hub and spoke keeps one line per rule in the context file and moves the detail to procedure documents the agent reads when a trigger fires. Path-scoped rules keep rules for one kind of file out of the window until the agent touches such a file. This page covers both, how they fail, and when to use which.

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

> A spoke that grows to thousands of lines is no easier to follow than a long context file. Give each spoke a size budget too, and split it when it grows.

## Path-scoped rules

### How they work

A path rule is a Markdown file in `.claude/rules/` with a `paths:` list of globs in its frontmatter. It loads only when a file tool reads or edits a matching file. A rule with no `paths:` loads at launch like a context file.

```markdown
---
paths: ["docs/**/*.md"]
---
- Every new page gets a "Sources" section with a verification date per source.
- No em dashes; use a comma or a full stop.
```

### When to use them

Use a path rule when the rule applies to one kind of file: tests, docs, migrations, infrastructure. It keeps those rules out of the window during unrelated work.

| Rule file | `paths:` | Loads when the agent touches |
|---|---|---|
| `tests.md` | `["**/tests/**", "**/*_test.go"]` | Any test file |
| `migrations.md` | `["**/migrations/**"]` | A migration |
| `infra.md` | `["infra/**", "**/*.tf"]` | Infrastructure code |
| `docs.md` | `["docs/**/*.md"]` | A documentation page |

### Limits

#### Shell-only work

Path rules load only when a file tool touches a matching path. An agent that writes a file with `cat > file` or a generator never sees them. If agents in your setup do much of their work through the shell, back the important path rules with a hook.

#### Compaction

A path rule that loaded earlier is summarised away at compaction and returns only on the next matching touch. See [Context files after compaction]({{ '/sections/04-context-files/after-compaction.html' | relative_url }}).

#### Codex

Codex does not document path-scoped rules. For a repository used with both clients, the nearest equivalent is an `AGENTS.md` in the folder that holds those files.

## Choosing between them

| The detail is | Use |
|---|---|
| A procedure for an occasional task (release, migration, incident) | Hub and spoke |
| Conventions for one kind of file, needed whenever such a file is edited | A path rule |
| Rules for one folder, entered through file reads | A subdirectory `CLAUDE.md` |
| A rule that must hold every time | Any of the above, plus a hook or check |

## Examples from my own harness

Each procedure document is named by one line in the shared context file, with the trigger for reading it. Path-scoped rules cover tests, documentation and skill authoring, so those rules appear only when an agent works on those files.

## Sources

- Path rules: <https://code.claude.com/docs/en/memory.md>, verified 2026-11-08; after compaction: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-09.
- Hub and spoke is from the author's own harness, described conceptually. Example files are illustrations written for this page.

{% include page_nav.html %}
