---
title: "Worktree isolation"
parent: "Agent profiles, sub-agents and delegation"
nav_order: 2
last_updated: "2026-11-09"
talk_stop: "6"
stop_minutes: "4"
---

{% include page_nav.html %}

# Worktree isolation

When several agents write to one repository at once, each needs its own checkout or they overwrite each other's work. A git worktree is that checkout: a second working directory of the same repository, on its own branch, sharing one object store. This page covers why worktrees, how to create one per agent, a worktree's lifecycle from creation to safe removal, merging back, and the ways isolation fails.

![A lifecycle with five main states from left to right: created, agent works, merged, audit, removed. From agent works, a merge failure leads to a conflict state that returns after a rebase and retest, and an early deletion leads to a work-lost failure state. From audit, a result that is not proved leads to a kept state, which returns to audit on the next pass.]({{ '/assets/images/08-delegation/worktree-lifecycle.png' | relative_url }}){: style="width:100%"}

*Figure 1. A worktree's life, with the two ways it goes wrong. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Why worktrees

### Two agents, one checkout

| What happens | Why it hurts |
|---|---|
| Agent A edits `parser.py`; agent B edits it a second later | One edit silently replaces the other |
| Both commit | One commit holds both agents' changes, so neither can be reviewed alone |
| A runs the tests while B is half-way through an edit | A sees failures it did not cause |
| B switches branch to check something | A's files change underneath it |

### What a worktree gives

<!-- IMAGE NEEDED: one git repository with several worktrees, each a separate directory on its own branch, sharing one object store; search: git worktree diagram, git worktrees explained -->

- Its own directory on disk.
- Its own branch, checked out only there.
- One shared repository history, so creating it is fast and it sees every branch.
- Changes that arrive as one branch, reviewable and mergeable on their own.

### Worktree, clone or container

| | Worktree | Separate clone | Container |
|---|---|---|---|
| Disk cost | Files only; history shared | Full copy | Full copy plus image |
| Setup time | Seconds | Longer for a big repository | Longest |
| Sees other branches at once | Yes, same repository | After a fetch | After a fetch |
| Isolates processes and network | No | No | Yes |

A worktree isolates files, not processes. If two agents' tests share a database or a port, they still collide; see [nesting and concurrency]({{ '/sections/08-delegation/nesting-and-concurrency.html' | relative_url }}#concurrency).

## Creating one per agent

### By hand

```bash
git fetch origin
git worktree add ../repo-agent-a -b agent-a/fix-parser origin/main
git worktree add ../repo-agent-b -b agent-b/add-tests   origin/main
git worktree list
```

Start each from a fresh `origin/main`, not from whatever the lead's checkout has, so the branch carries only the agent's own work.

### From a profile

Claude Code's `isolation: worktree` profile setting does it per launch.

```markdown
---
name: engineer
tools: Read, Grep, Glob, Edit, Write, Bash
isolation: worktree
---
```

- The worktree is temporary, branched by default from the default branch instead of the parent session's current commit.
- If the agent makes no changes, the worktree is removed automatically.

### Every repository the agent writes to

1. An agent is briefed to change a library and the service that uses it.
2. It has a worktree only in the library's repository.
3. Its edits to the service land in the service's main checkout, where other sessions work.

The fix is a worktree in each repository it writes to, and a launch guard that refuses a writing agent without one.

## The lifecycle

### Created

- A branch off fresh `origin/main`, in its own directory.
- Nothing in it yet but the base commit.

### Agent works

- The agent edits, runs tests and commits each logical step.
- A process runs inside the worktree for as long as the agent does.

### Merged

- The lead or a person reviews the branch.
- It is merged into main, or sent back with findings.

#### Conflict

If main moved while the agent worked, the merge can conflict. The fix happens on the branch:

1. Rebase the branch onto the new main.
2. Resolve the conflict in the agent's worktree, by the agent that knows the change.
3. Rerun the tests.
4. Try the merge again.

### Audit

Two questions, both answered with evidence before removal:

| Question | Check |
|---|---|
| Did all of its content reach main? | Every commit or its content is on main |
| Is any process still running in it? | No live process has the directory as its working directory |

### Removed

Only when both answers are proved: the worktree directory and its branch are removed.

```bash
git worktree remove ../repo-agent-a
git branch -d agent-a/fix-parser   # -d refuses an unmerged branch
```

> Warning: removing a worktree before its work is merged deletes that work. "Unproved" means kept, never removed. Check the content against main, not just whether the branch name looks old.

## Merging back

### One branch at a time

| Merging one branch at a time | Merging all at once |
|---|---|
| A failure holds up only its own branch | One failure blocks every change |
| A bad change reverts as one unit | Reverting separates tangled changes by hand |
| A conflict names one agent's change | A conflict mixes several agents' changes |

### Worked example: two branches, one conflict

```text
lead$ git merge --no-ff agent-a/fix-parser
Merge made by the 'ort' strategy.
lead$ git merge --no-ff agent-b/add-tests
CONFLICT (content): Merge conflict in tests/test_parser.py
lead$ git merge --abort
lead> Engineer B: main moved. Rebase agent-b/add-tests onto main, rerun tests, report.
```

The lead does not resolve B's conflict in its own checkout. It hands it back to the agent that knows the change.

## Commands and edge cases

### The `git worktree` commands

| Command | What it does |
|---|---|
| `git worktree add <path> -b <branch> <base>` | Creates a worktree at `<path>` on a new branch from `<base>` |
| `git worktree list --porcelain` | Lists every worktree with its path, commit and branch, in a form scripts can parse |
| `git worktree remove <path>` | Removes a worktree; refuses if it has uncommitted changes or untracked files, unless `--force` |
| `git worktree prune` | Clears records of worktrees whose directories were deleted by hand |
| `git worktree lock <path>` | Protects a worktree from `prune`, for example one on a removable drive |
| `git worktree move <path> <new-path>` | Moves a worktree to another directory |

### Edge cases

| Case | What happens | What to do |
|---|---|---|
| Checking out a branch that another worktree already has | Git refuses: one branch can be checked out in one worktree at a time | Give each agent its own branch |
| A worktree directory deleted with `rm` | Git still lists it until pruned | `git worktree prune`, and remove worktrees with `git worktree remove` instead |
| Dependencies and build output | Each worktree has its own files, so a virtual environment or `node_modules` is not shared | Install dependencies per worktree, or point each at a shared cache |
| A branch merged by squash | Its commits are not ancestors of main, so `git branch --merged` does not list it | Compare content against main before calling it merged |
| Uncommitted work at removal | `git worktree remove` refuses | Commit or discard it on purpose; `--force` only after checking what it deletes |

## How isolation fails

| Failure | What it looks like | Check |
|---|---|---|
| Commit on the parent's branch | The sub-agent's commits appear on the lead's branch | `git log` on both branches after the run |
| Second repository unprotected | Writes land in another repository's main checkout | Give a worktree in every repository it writes to |
| Shared test resources | Tests fail only when two agents run at once | Separate databases or ports per worktree, or bound concurrency |
| Early removal | A branch's commits exist nowhere else | Audit before removal |
| Stale base | The branch carries old main changes | Create from fresh `origin/main` |

## Examples from my own harness

- A launch guard refuses to start a writing sub-agent without its own worktree.
- A worktree is removed only after an audit proves its content reached the main branch.
  - A live process inside it blocks removal.
- Worktrees that fail the audit are kept and checked again on the next pass.

## Related pages

- [Agent profiles]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}), where `isolation: worktree` is set.
- [Checking delegated work]({{ '/sections/08-delegation/checking-delegated-work.html' | relative_url }}), what to check before merging a branch.
- [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}), when the writers are sessions, not sub-agents.

## Sources

- `git worktree` reference: <https://git-scm.com/docs/git-worktree>.
- `isolation: worktree`: <https://code.claude.com/docs/en/sub-agents.md>, verified 2026-11-09.
- The launch guard and removal audit are from the author's own harness, described conceptually. The examples are illustrations written for this page.

{% include page_nav.html %}
