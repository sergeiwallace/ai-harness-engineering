---
title: "Checking delegated work"
parent: "Agent profiles, sub-agents and delegation"
nav_order: 4
redirect_from:
  - /sections/08-delegation/detail.html
  - /sections/08-delegation/advanced.html
last_updated: "2026-11-09"
talk_stop: "6"
stop_minutes: "4"
---

{% include page_nav.html %}

# Checking delegated work

A sub-agent's report describes what the sub-agent believes it did. Agents report work they did not do, and summaries drop the caveat that changes the conclusion. The lead's job at collection time is to check the artifact the report describes: the commits, the diff, the test output, the source passage. This page covers the two failure modes, a checking routine you can run on every report, worked examples, and how to make reports easier to check.

![A sequence across five participants. A sub-agent reports that it fixed a race and all 214 tests pass. The lead runs git log and diff and finds no new commit and no change, so it returns the report with the evidence. On the second attempt the sub-agent reports a commit and quotes a test line; the lead shows the commit, sees two files changed, runs the quoted command and gets 214 passed, and accepts. In a third exchange the sub-agent reports that a library supports streaming; the lead reads the cited passage and finds it is experimental and off by default.]({{ '/assets/images/08-delegation/check-report.png' | relative_url }}){: style="width:100%"}

*Figure 1. Three reports and the checks that settle them. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Two ways a report misleads

### Work reported but not done

An agent can return a complete, confident report while nothing changed.

- A fix report with passing tests, and an empty diff.
- "Filed two records", and no records exist.
- "Updated the docs", and the commit touches only code.

The report describes the plan as if it had been carried out.

### The caveat that dropped out

A summary keeps the headline and loses the qualifier. "The API supports batching" without "only on the paid tier". "The library streams responses" without "experimental and off by default". The dropped part is often the one that changes the plan.

| | Work not done | Caveat dropped |
|---|---|---|
| Typical task | Code change, bookkeeping | Research, summary |
| What the report says | Done, tests pass | A clean yes |
| What is true | Nothing changed | Yes, with a condition |
| The check | Commits, diff, test output | The cited passage |

## The checking routine

### For a code change

<!-- IMAGE NEEDED: a terminal showing git log --oneline and git diff --stat output for a branch; search: git diff stat terminal screenshot, git log oneline terminal -->

1. Commits exist where expected: `git -C <worktree> log --oneline <base>..HEAD`.
2. Files changed: `git -C <worktree> diff --stat <base>..HEAD`.
3. The change matches the brief: read the diff, not the summary of it.
4. Tests ran: the report quotes the exact command and its last line. Rerun it if the result matters.
5. Nothing landed elsewhere: the lead's own branch and main have no unexpected commits.

### For research

1. Each claim the decision rests on has a cited source.
2. Open the source and read the passage, not just the page title.
3. Look for conditions next to the claim: tier, flag, version, "experimental".

### When a check fails

Send the report back with the evidence: the command and its output. Do not fix it yourself in the sub-agent's worktree; the sub-agent knows the change, and the next report will show whether it understood.

<details markdown="1">
<summary>A decision path for every report</summary>

| Question | If no | If yes |
|---|---|---|
| Did any file change? | Send back with the empty diff | Next question |
| Are the commits on the agent's branch? | Find where they landed; fix isolation | Next question |
| Is the test command and its output in the report? | Run the tests yourself | Next question |
| Does the diff match the brief? | Send back with the mismatch | Next question |
| Does a decision rest on a cited source? | Accept | Read the key passage, then accept |

</details>

## Automating the checks

The routine above works when the lead remembers to run it. In the systems approach it is a step in the collection workflow, so it runs on every report. The code-change checks fit in a few lines of shell:

```bash
# collect.sh <worktree> <report-file>: refuse a report the artifacts do not back
base=origin/main; wt="$1"; report="$2"
ahead=$(git -C "$wt" rev-list --count "$base"..HEAD)
[ "$ahead" -gt 0 ] || { echo "refused: no commits on the branch"; exit 1; }
git -C "$wt" diff --quiet "$base"...HEAD && { echo "refused: empty diff"; exit 1; }
grep -Eq '[0-9]+ passed' "$report" || { echo "refused: no test output quoted"; exit 1; }
echo "artifacts present: now read the diff against the brief"
```

1. The sub-agent finishes and writes its report.
2. The workflow runs the collection script before the lead reads the report.
3. A refusal goes back to the sub-agent with the script's message as the evidence.
4. A pass means the artifacts exist; the lead still reads the diff against the brief and reruns the tests where the result matters.

| Check | Automated | Left to the lead |
|---|---|---|
| Commits exist on the branch | Yes | |
| The diff is not empty | Yes | |
| The report quotes test output | Yes | |
| The diff matches the brief | | Yes |
| A cited passage says what the summary says | | Yes |

## Worked examples

### A fix with no work

```text
sub-agent> Fixed the race in the retry loop; all 214 tests pass.
lead$ git -C ../repo-agent-a log --oneline origin/main..HEAD
                                         <- no commits on the branch
lead$ git -C ../repo-agent-a status --short
                                         <- no uncommitted changes either
lead> Returned: your branch has no commits and no changes. Please make the fix,
      commit it, and quote the test command with its last output line.
```

### The second attempt

```text
sub-agent> Commit 4e1c: hold the lock across the retry. `pytest tests/retry -q`: 214 passed.
lead$ git -C ../repo-agent-a show --stat 4e1c
 src/retry.py          | 9 ++++++---
 tests/test_retry.py   | 21 +++++++++++++++++++++
lead$ pytest tests/retry -q
214 passed in 6.1s
lead> Accepted.
```

### A dropped caveat

1. A research sub-agent reports "the library supports streaming responses".
2. The lead opens the cited page.
3. The page says "streaming is experimental and disabled by default".
4. The plan changes before any code is written.
   - Either the design enables the flag on purpose and records the risk.
   - Or it does without streaming.

## Making reports easy to check

The brief decides how checkable the report will be. Ask for evidence in a fixed shape, and the check becomes a quick comparison instead of an investigation.

| Ask for | Instead of |
|---|---|
| Commit ids | "Committed the fix" |
| The exact test command and its last output line | "Tests pass" |
| The file and line of each finding | "There are a few issues in the parser" |
| A quote of the source passage with its link | "The docs say it is supported" |
| "Partial" with what is left, when time ran out | A report that reads complete |

The [brief's Output field]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}#output) is where these go.

## Why this happens

### The report comes from the agent's own context

- A sub-agent writes its report from what is in its context.
- If its context holds a detailed plan and a belief that the plan ran, the report reads as if it ran.
- Nothing in writing the report forces a look at the files.
- The fix is in the workflow: the lead's collection step runs the checks above every time, and a brief's Output field asks for evidence the lead can check.

### Summaries compress

A summary has a length budget, and the qualifier is the first thing to go because the headline still reads as true without it.

> Important: check the artifact the report describes: the commits, the diff, the test output or the cited passage.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Accepting "tests pass" without output | Require the command and its last line |
| Fixing the sub-agent's work yourself | Send it back with evidence |
| Reading only the summary of research a decision rests on | Read the key passage |
| Checking only that a branch exists | Check its diff against the brief |

## Related pages

- [Sub-agent briefs]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}), where the output format is set.
- [Worktree isolation]({{ '/sections/08-delegation/worktree-isolation.html' | relative_url }}), where commits should land.
- [Testing]({{ '/sections/12-testing/' | relative_url }}) and [Code review]({{ '/sections/13-code-review/' | relative_url }}), the checks that run before merge.

## Sources

- The checking routine and the false-report and dropped-caveat cases are from the author's own harness, described conceptually. The transcripts are illustrations written for this page.

{% include page_nav.html %}
