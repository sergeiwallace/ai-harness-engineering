---
title: "Compiled shared context files"
parent: "Context files"
nav_order: 4
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# Compiled shared context files

Once you work across several repositories, machines or clients, one hand-edited context file stops being enough. The same rules need to reach every session, with a few lines that differ per client and per machine, and the shared file needs a limit so it does not grow with every lesson. A compiled context file solves both: small source parts are joined by a script, checked against a size budget, written to a versioned directory and linked into place. This page describes the pattern as it works in my own harness, at the level of mechanism.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## When you need it

| Situation | One hand-written file per repository is enough | Compile |
|---|---|---|
| One machine, one client | Yes | |
| The same lines copied into every repository | | Yes |
| Two clients that need slightly different files | | Yes |
| Several machines with different tools and traps | | Yes |

To start: one hand-written `CLAUDE.md` per repository. Add a user-level file when you notice the same lines in every repository. Compile when you have two clients or two machines to keep in step.

### Built-in options to try first

Claude Code already has ways to share instructions without a build step. Try them before writing a compiler.

| Need | Built-in option | Limit |
|---|---|---|
| The same personal lines in every project | `~/.claude/CLAUDE.md` and `~/.claude/rules/` | One machine, one client |
| One file shared across projects or worktrees | `@~/path/shared.md` import from each project's `CLAUDE.md` | Needs approval once per project; no size gate |
| A shared set of rules | Symlink the files into each project's `.claude/rules/` | Links outside the project need approval; no size gate |
| Organization-wide instructions | A managed `CLAUDE.md` (or the `claudeMd` managed setting) deployed by IT | Same text everywhere; no per-machine part |
| One file for Claude Code and Codex | `AGENTS.md`, imported from a short `CLAUDE.md` | No per-client part beyond the importing file |

Compile when you need what none of these give: a per-client and per-machine part, a size budget that fails the build, and a rollback.

## The pattern

### One change, start to finish

![Sequence: an author adds a lesson, the compiler fails the build over budget, the author moves the detail to a spoke, the build passes and is linked into every client; later a bad edit is rolled back by relinking the previous version]({{ '/assets/images/04-context-files/compiled-file-sequence.svg' | relative_url }})

*Adding one lesson to the compiled shared file. The size gate refuses the first try, so nothing is installed; the second try moves the detail to a spoke. Rollback is a relink. Diagram made with archify.*

### The parts

| Part | Holds | Example line (illustrative) |
|---|---|---|
| Base | Rules for every session everywhere | "Every bug fix starts with a failing test." |
| Client | Lines for one client's features and names | "Use `/goal` for unattended runs." |
| Machine | One machine's paths, tools and traps | "The system Python is old; use the managed one." |

The compiler joins base, the client's part and the machine's part, in that order, into one file per client and machine.

### The size gate

The compiler measures the joined file against a budget and fails the build when it is over. In my harness, before the size check existed, the shared file grew every time a lesson was added.

```text
$ compile
  base 9.1 KB + client 2.4 KB + machine 1.8 KB = 13.3 KB
  budget 12.0 KB: FAIL, over by 1.3 KB
  nothing installed
```

The output is illustrative. A failed build installs nothing, so an oversized file never reaches a session.

### Versioned install

Each passing build is written to a new directory. The client's config location is a link to the current one.

```text
generations/
├── 0041/   CLAUDE.md  AGENTS.md
├── 0042/   CLAUDE.md  AGENTS.md
└── current -> 0042
```

Every session that starts after the link moves reads the new file. Rolling back means pointing the link at the previous directory; nothing is rebuilt.

### Install on every machine

1. Each machine runs the same install step.
2. The step builds the file from the base, the client's part and that machine's part.
3. A change to the base reaches every machine at its next install.
   - A change to one machine's part reaches only that machine.
4. Each machine keeps its own versioned directories, so a rollback on one machine leaves the others alone.

## Building a small one

### The layout

```text
context/
├── base.md                 rules for every session
├── client/
│   ├── claude-code.md      Claude Code only
│   └── codex.md            Codex only
├── machine/
│   ├── laptop.md
│   └── build-server.md
└── compile.py
```

### The compiler

This is a complete, minimal version of the pattern: join, measure, refuse or write a new version, move the link.

```python
#!/usr/bin/env python3
"""Join context parts, enforce a size budget, install as a new version."""
import pathlib, sys, time

BUDGET = 12_000  # bytes; pick yours and keep it
SRC = pathlib.Path(__file__).parent
OUT = pathlib.Path.home() / ".local/state/context/generations"

def build(client: str, machine: str) -> str:
    parts = [SRC / "base.md", SRC / "client" / f"{client}.md", SRC / "machine" / f"{machine}.md"]
    return "\n\n".join(p.read_text() for p in parts)

def main(client: str, machine: str, target: str) -> int:
    text = build(client, machine)
    size = len(text.encode())
    if size > BUDGET:
        print(f"{size} bytes, budget {BUDGET}: FAIL, nothing installed", file=sys.stderr)
        return 1
    gen = OUT / time.strftime("%Y%m%d-%H%M%S")
    gen.mkdir(parents=True)
    (gen / pathlib.Path(target).name).write_text(text)
    current = OUT / "current"
    tmp = OUT / "current.new"
    tmp.unlink(missing_ok=True)
    tmp.symlink_to(gen)
    tmp.replace(current)  # atomic switch to the new version
    print(f"{size} bytes, installed {gen.name}")
    return 0

if __name__ == "__main__":
    sys.exit(main(*sys.argv[1:4]))
```

Link the client's file to the current version once, for example `~/.claude/CLAUDE.md` to `generations/current/CLAUDE.md`, and every later build reaches it without touching the client's directory again. The script is written for this page; adapt the paths to your clients.

### Rolling back

```bash
cd ~/.local/state/context/generations
ls                                  # 20261108-091500  20261109-101200  current
ln -sfn 20261108-091500 current     # the next session reads the previous file
```

## Rules that keep it working

1. Edit the source parts, never the output; a direct edit is lost at the next build.
2. Build from the committed parts, so every machine can rebuild the same file.
3. Keep the budget, and move detail to spokes when it is reached.
4. Keep machine parts to facts about that machine; anything true everywhere belongs in the base.

<details markdown="1">
<summary>Why the output must never link into a temporary checkout</summary>

If the client's config location links to a file inside a temporary working copy, removing that copy breaks every session's context at once. Render the output from the main checkout into its own versioned directory, and link only to that.

</details>

## Trade-offs

| Gain | Cost |
|---|---|
| One edit reaches every client and machine | A build step between editing and seeing the change |
| A size budget that is enforced | Lessons must be compressed or moved to spokes |
| One-step rollback | A directory per build to keep or clean up |

## Sources

- The pattern is from the author's own harness, described conceptually. Commands, sizes and directory names are illustrations written for this page.
- Built-in sharing options (user files and rules, imports, symlinked rules, managed `CLAUDE.md`, `AGENTS.md`): [Claude Code: how Claude remembers your project](https://code.claude.com/docs/en/memory.md), read 2026-11-09.

{% include page_nav.html %}
