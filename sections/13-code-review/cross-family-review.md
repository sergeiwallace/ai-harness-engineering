---
title: "Cross-family review"
parent: "Code review"
nav_order: 1
redirect_from:
  - /sections/13-code-review/detail.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Cross-family review
{: .no_toc }

Cross-family review means the agent that reviews a change comes from a different model family than the agent that wrote it: a Codex model reviews Claude's code, and a Claude model reviews Codex's. Models from one family share training data, training methods and habits, so they tend to share blind spots and to rate each other's output favourably.

This page covers:

1. the evidence on self-preference, and how reviewer types compare;
2. how to run a cross-family review from either side, with the commands;
3. how to write the review brief;
4. what to do when the reviewer and the author disagree;
5. what cross-family review does not fix, and its pitfalls.

<figure>
  <img src="{{ '/assets/images/13-code-review/cross-family.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: launcher, author (a Claude model), branch, and reviewer (a Codex model). The launcher gives the author a task with acceptance criteria; the author commits code with passing tests. The launcher sends the reviewer a brief with the diff, the criteria and the line that no change is a valid result; the reviewer never sees the author's reasoning. The reviewer reads every changed hunk and returns three findings with file and line. The launcher has the author fix two real ones and records one as refuted. The author commits the fixes, and the launcher merges and records which family reviewed.">
  <figcaption>One delegated change, written by one family and reviewed by another. Made with archify.</figcaption>
</figure>

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## Model family and review

### Self-preference

Self-preference is a model's tendency to rate its own output above equal-quality output from others. The evidence:

| Study | Finding |
|:--|:--|
| NeurIPS 2024 (GPT-4, Llama 2) | Models recognise their own outputs at better than chance |
| Same study | They score their own outputs higher than others' when human raters judge them equal |
| Same study | The stronger the self-recognition, the stronger the self-preference; controlled experiments pointed to a causal link |
| 2026, 20 models | Capability did not predict low self-preference |
| Same 2026 study | Judging on several separate dimensions reduced measured bias by 31.5% |

The last row is a second lever besides changing the family: ask the reviewer to judge named criteria one at a time, which the [review brief](#writing-the-review-brief) below does.

### Reviewer types compared

A same-family reviewer in a fresh context does not see the author's reasoning. It still shares the author's training, so it is a weaker check than a reviewer from another family.

| Reviewer | Shares the author's reasoning? | Shares the author's training? | Strength |
|:--|:--|:--|:--|
| The author re-reading its own work | Yes | Yes | Weakest; prone to confirming itself |
| Same family, fresh context | No | Yes | Useful; shares blind spots |
| Other family, fresh context | No | No | Strongest agent review |
| A person with domain knowledge | No | No | Strongest for intent and product judgment |

#### Fresh context

Claude Code's best-practices page describes the shape: "A reviewer running in a fresh subagent context sees only the diff and the criteria you give it, not the reasoning that produced the change." Two things follow:

- The reviewer judges the result, without the author's explanation of why it is right.
- The reviewer needs everything it should use in the brief, because it has nothing else.

## Running a cross-family review

### From the Claude side

Claude Code's own review commands, `/review` and `/security-review`, run in a separate context but use a Claude model. To get a reviewer from the other family, call Codex's review command from the Claude session:

```bash
# Review the branch against main with Codex's built-in review
codex review --base main

# Or review with your own brief instead (a custom prompt cannot be combined with --base)
codex review - < review/brief.md
```

`codex review` options, from OpenAI's Codex CLI reference:

| Option | What it does |
|:--|:--|
| `--base <branch>` | "Review changes against the specified base branch." |
| `--commit <SHA>` | "Review the changes introduced by the specified commit." |
| `--uncommitted` | "Review staged, unstaged, and untracked changes." |
| `PROMPT`, or `-` for standard input | "Custom review instructions." |

The reference states that `--uncommitted`, `--base`, `--commit` and a custom prompt conflict with one another, so a custom brief has to name the diff itself (for example, by including `git diff origin/main...HEAD` output or a path to it).

### From the Codex side

The same pattern in reverse: a Codex session asks Claude Code for a review in non-interactive mode, limited to read-only tools.

```bash
git diff origin/main...HEAD > review/diff.patch
claude -p --allowedTools "Read,Grep,Glob" < review/brief.md
```

- `-p` runs Claude Code once, prints the result and exits.
- `--allowedTools` limits the reviewer to reading and searching, so it cannot change the code it reviews.

### Record the family

To record who wrote a change and who reviewed it, commit trailers are enough:

```text
Authored-by-model: claude-opus
Reviewed-by-family: codex
```

With the family recorded, two numbers become countable:

- the share of merged changes reviewed across families;
- how many of each family's findings turned out real after [verification]({{ '/sections/13-code-review/verifying-findings.html' | relative_url }}).

## Writing the review brief

### The four parts

1. The scope: the diff, and nothing outside it unless the reviewer needs callers for context.
2. The criteria the change was written against.
3. An output format: file and line, what is wrong, why it matters, how to verify, and a severity.
4. Permission to find nothing: "No change warranted" is a valid result.

### Worked example

```text
You are reviewing a change written by another model. You have not seen its reasoning.

Scope: review/diff.patch. Read callers of changed functions if you need them.
Criteria: review/criteria.md (AC1 to AC4).

For each finding: file:line, the problem, why it matters against a criterion, and a
check that would confirm it (a command, a test, a trace). Sort into:
  must-fix       breaks a criterion or is unsafe
  worth a look   a real risk, not certain
  optional       style or preference

"No change warranted" is a valid and useful result. Do not invent findings.
```

## When the reviewer and the author disagree

A cross-family reviewer will sometimes be wrong, and the author will sometimes argue a real finding away. A path that settles it with evidence:

1. The author answers each must-fix finding with "fixed" or "disputed", and a reason for a dispute.
2. A disputed finding goes to a [verifier]({{ '/sections/13-code-review/verifying-findings.html' | relative_url }}), or to the owner, with the reviewer's confirming check.
3. The verifier runs the check (a test, a trace) and reports confirmed or refuted.
4. The owner rules; a refuted finding is recorded so the next round does not raise it again.

The author does not settle its own disputes: it holds the reasoning that produced the code.

## What cross-family review does not fix

| Limit | Why | What covers it |
|:--|:--|:--|
| Missing context | A reviewer that cannot see a caller in another repository cannot judge it | Give it the other repositories, or a contract test |
| Code that should have changed but did not | It is not in the diff | Compare the changed files to the task; ask what is missing |
| Runtime and security behaviour | Reading is not running | Tests, scans, a [security review]({{ '/sections/13-code-review/#security-review' | relative_url }}) |
| Product intent | Neither model knows what the user wanted beyond the criteria | A person |

Anthropic's April 2026 postmortem is the clearest example of the first limit:

- A defect passed several human and automated reviews, tests and internal use.
- A stronger reviewer caught it only when given the additional repositories it needed for full context.

## Pitfalls

| Pitfall | Why it bites | Fix |
|:--|:--|:--|
| Running `/review` and calling it cross-family | It uses the same family as a Claude author | Call the other family's tool, such as `codex review` |
| A brief with no criteria | The reviewer invents its own standard | Put the acceptance criteria in the brief |
| A reviewer with write access | It "fixes" what it reviews, and nobody reviews that | Read-only tools for the reviewer |
| Only one family available on your provider | Cross-family review is not possible | Same-family in a fresh context, recorded as weaker |
| Counting reviews, not real findings | A busy reviewer looks useful | Record how many findings survived verification |

## From my harness

How review runs in my harness:

1. The author is usually a Claude model; the reviewer is a Codex model, and the reverse when Codex wrote the change.
2. Each commit records which model and family did the work and which reviewed it.
3. Cross-family review is preferred and never blocks a merge.
   - When no other family is available, a same-family review in a fresh context is accepted and recorded as weaker.
4. Every review brief carries the "no change warranted is a valid result" line.

Steps 1 and 2 are built into the review workflow and its commit tool (layer 2 of the harness); the brief line in step 4 is layer 1.

## Sources

- [Self-preference in LLM evaluators](https://papers.nips.cc/paper_files/paper/2024/hash/7f1f0218e45f5414c79c0679633e47bc-Abstract-Conference.html), verified 2026-07-26.
- [Self-preference across 20 models](https://arxiv.org/abs/2604.22891), verified 2026-09-20.
- [Claude Code best practices](https://code.claude.com/docs/en/best-practices), verified 2026-10-08.
- [Claude Code commands (`/review`, `/security-review`)](https://code.claude.com/docs/en/commands.md), verified 2026-11-08.
- [Codex CLI command reference (`codex review`)](https://learn.chatgpt.com/docs/developer-commands?surface=cli), verified 2026-11-09.
- [Claude Code April 2026 postmortem](https://www.anthropic.com/engineering/april-23-postmortem), verified 2026-07-26.

{% include page_nav.html %}
