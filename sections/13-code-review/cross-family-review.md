---
title: "Cross-family review"
parent: "Code review"
nav_order: 1
redirect_from:
  - /sections/13-code-review/detail.html
---

{% include page_nav.html %}

# Cross-family review
{: .no_toc }

As of 2026-11-09. Part of [Code review]({{ '/sections/13-code-review/' | relative_url }}).

Cross-family review means the agent that reviews a change comes from a different model family than the agent that wrote it: a Codex model reviews Claude's code, and a Claude model reviews Codex's. Models from one family share training data, training methods and habits, so they tend to share blind spots and to rate each other's output favourably. A reviewer from another family has different blind spots. This page covers the evidence, how to run it in practice, how to write the review brief, and what it does not fix.

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

The last row is a second lever besides changing the family: ask the reviewer to judge named criteria one at a time, which the four-part brief below does.

### Same-family review

A same-family reviewer in a fresh context does not see the author's reasoning. It still shares the author's training, so it is a weaker check than a reviewer from another family.

| Reviewer | Shares the author's reasoning? | Shares the author's training? | Strength |
|:--|:--|:--|:--|
| The author re-reading its own work | Yes | Yes | Weakest; prone to confirming itself |
| Same family, fresh context | No | Yes | Useful; shares blind spots |
| Other family, fresh context | No | No | Strongest agent review |
| A person with domain knowledge | No | No | Strongest for intent and product judgment |

#### Fresh context

Claude Code's best-practices page describes the shape: "A reviewer running in a fresh subagent context sees only the diff and the criteria you give it, not the reasoning that produced the change." The reviewer judges the result, not the story told about it.

## Running a cross-family review

### From the Claude side

Claude Code's `/review` command reviews a branch or pull request in a separate context, and `/security-review` looks for security risks. To get a reviewer from the other family, call the other family's tool from the Claude session, with the diff and the criteria as files:

```bash
git diff origin/main...HEAD > /tmp/review/diff.patch
codex exec --sandbox read-only "$(cat /tmp/review/brief.md)"
```

### From the Codex side

The same pattern in reverse: a Codex session delegates review to Claude Code in non-interactive mode, with read-only tools.

```bash
claude -p --allowedTools "Read,Grep,Glob" "$(cat /tmp/review/brief.md)"
```

### Record the family

To record who wrote a change and who reviewed it, a commit trailer is enough:

```text
Reviewed-by-family: codex
Authored-by-model: claude-opus
```

With the family recorded, you can count what share of merged changes were reviewed across families, and how many of each family's findings turned out real.

## Writing the review brief

### The four parts

1. The scope: the diff, and nothing outside it unless the reviewer needs callers for context.
2. The criteria the change was written against.
3. An output format: file and line, what is wrong, why it matters, how to verify, and a severity.
4. Permission to find nothing: "No change warranted" is a valid result.

### Worked example

```text
You are reviewing a change written by another model. You have not seen its reasoning.

Scope: diff.patch. Read callers of changed functions if you need them.
Criteria: criteria.md (AC1 to AC4).

For each finding: file:line, the problem, why it matters against a criterion, and a
check that would confirm it (a command, a test, a trace). Sort into:
  must-fix       breaks a criterion or is unsafe
  worth a look   a real risk, not certain
  optional       style or preference

"No change warranted" is a valid and useful result. Do not invent findings.
```

> Anthropic's prompting guidance warns that newer models over-react to aggressive wording and tend to over-engineer, including defensive code. A brief that only says "find problems" invites invented ones. See [Review rounds and stop rules]({{ '/sections/13-code-review/review-loops.html' | relative_url }}).

## What cross-family review does not fix

| Limit | Why | What covers it |
|:--|:--|:--|
| Missing context | A reviewer that cannot see a caller in another repository cannot judge it | Give it the other repositories, or a contract test |
| Code that should have changed but did not | It is not in the diff | Compare the changed files to the task; ask what is missing |
| Runtime and security behaviour | Reading is not running | Tests, scans, a security review |
| Product intent | Neither model knows what the user wanted beyond the criteria | A person |

Anthropic's April 2026 postmortem is the clearest example of the first limit. A defect passed several human and automated reviews; a stronger reviewer caught it only when given the additional repositories it needed for full context.

## From my harness

How review runs in my harness:

1. The author is usually a Claude model; the reviewer is a Codex model, and the reverse when Codex wrote the change.
2. Each commit records which model and family did the work and which reviewed it.
3. Cross-family review is preferred and never blocks a merge.
   - When no other family is available, a same-family review in a fresh context is accepted and recorded as weaker.
4. Every review brief carries the "no change warranted is a valid result" line.

Steps 1 and 2 are built into the review workflow and its commit tool (layer 2 of the harness); the brief line in step 4 is layer 1.

## Sources

- Self-preference in LLM evaluators: <https://papers.nips.cc/paper_files/paper/2024/hash/7f1f0218e45f5414c79c0679633e47bc-Abstract-Conference.html>, verified 2026-07-26.
- Self-preference across 20 models: <https://arxiv.org/abs/2604.22891>, verified 2026-09-20.
- Claude Code best practices: <https://code.claude.com/docs/en/best-practices>, verified 2026-10-08.
- Claude Code commands (`/review`, `/security-review`): <https://code.claude.com/docs/en/commands.md>, verified 2026-11-08.
- Claude Code April 2026 postmortem: <https://www.anthropic.com/engineering/april-23-postmortem>, verified 2026-07-26.

{% include page_nav.html %}
