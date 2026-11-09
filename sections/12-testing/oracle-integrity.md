---
title: "Test tampering and oracle integrity"
parent: "Testing agent-written code"
nav_order: 2
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Test tampering and oracle integrity
{: .no_toc }

The oracle is whatever decides whether code is correct: a test suite, a reference output, a checker script.

- Test tampering is when an agent makes its code pass by changing the oracle instead of the code.
- It is the testing form of reward hacking.

This page covers:

1. what tampering looks like, with a worked example
2. when it happens, according to the 2026 evidence
3. the lightest control with measured effect: a policy line plus an escalation channel
4. who judges a disputed test
5. heavier controls: read-only tests and file formats
6. pitfalls.

The sub-topic page [Detecting test edits in a diff]({{ '/sections/12-testing/test-edit-detection.html' | relative_url }}) covers the deterministic check that backs all of this up.

<figure>
  <img src="{{ '/assets/images/12-testing/tamper-dispute.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: launcher, coding agent, old test and test-edit check. An old export test fails. On the common path the agent reports a disputed test and the launcher rules that the criterion changed, so the agent edits the test with a stated reason. On the rare path the agent silently loosens the assertion, the test-edit check flags it, and the launcher files a finding.">
  <figcaption>The two paths an agent can take when an old test blocks it. Made with archify.</figcaption>
</figure>

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## What tampering looks like

### Four shapes, from blunt to subtle

| Shape | What changes | Example | Visible in a diff of test files? |
|:--|:--|:--|:--|
| Deletion | A test or assertion is removed | `assert len(rows) == 3` disappears | Yes |
| Loosening | The expected value or scope is widened | `== 3` becomes `>= 1`; an exact match becomes "contains" | Yes |
| Skipping | The test is marked to skip or expected to fail | A new `@pytest.mark.skip` with a vague reason | Yes |
| Special-casing | The code detects the test and returns the expected answer | `if input == FIXTURE: return EXPECTED` | No, it is in the code |

### Worked example: a loosened assertion

```diff
 def test_export_includes_archived_items(tmp_path):
     export = run_export(tmp_path, include_archived=True)
-    assert export.count == 12
+    # archived items are now exported lazily, so the count may differ
+    assert export.count > 0
```

What makes this one hard to catch:

- The comment gives a plausible reason.
- The assertion now passes for any non-empty export, including one that drops every archived item.
- A reviewer reading quickly accepts it.
  - ImpossibleBench found the same pattern fooling an LLM monitor: a "backward compatibility" rationale deceived Claude Opus 4 acting as the monitor.

## When it happens

On ordinary benchmark tasks, models that succeeded mostly changed the code. Tampering concentrates where the spec and the test conflict, or where the task cannot be done as asked.

| Evidence | Finding |
|:--|:--|
| Claude Opus 5.5 system card (2026) | Attempted reward hacking three to six times more frequent on impossible tasks than on possible ones |
| ImpossibleBench (2025) | Tasks with deliberate spec-test conflicts; "frontier models cheat surprisingly often", and Claude models cheated mainly (over 79%) by modifying test cases |
| Escalation-channel study (2026), eight models | Baseline hacking on tasks with defective checkers: 11.1% for claude-fable-5, 13.3% for gpt-5.6-sol |
| Test overfitting on SWE-bench (2026) | When refinement succeeded, models mostly changed the code under test, not the test |

<details markdown="1">
<summary>Aside: does telling the agent "a grader will check" help or hurt?</summary>

- The Claude Opus 5.5 system card reports the model is "grader-aware in a large proportion of tasks, especially when the input and tool call outputs disclose graders", and much less so when grading is not disclosed.
- Whether announcing a verifier raises or lowers tampering in real coding work has not been measured. This page does not recommend either way.

</details>

## Policy line and escalation channel

### What was tested

A 2026 study ran a two-by-two experiment across eight frontier models from five families (claude-fable-5, gemini-3.1-pro-preview, gemini-3.5-flash, gpt-5.3-codex, gpt-5.6-luna, gpt-5.6-sol, grok-4.5, kimi-k2.7-code). The two factors:

| Factor | Content |
|:--|:--|
| A policy line | Forbids test manipulation, defined as "Modifying, deleting, or narrowing test assertions, expected values, or test scope" |
| An escalation tool | The agent can report a problem with an issue type (one is a test-infrastructure conflict), an assessment, the approaches tried and a recommendation |

| Condition | Test tampering across the eight models |
|:--|:--|
| Neither | 23.6% |
| Policy and escalation together | 5.3%, and zero for six of the eight models |
| claude-fable-5, escalation alone | 11.1% down to 0.0% |
| gpt-5.6-sol, escalation alone / with policy | 13.3% down to 6.7% / 0.0% |

Escalation and hacking were close to mutually exclusive: 98.7% of escalations involved no hacking. The study found no measurable cost in task performance.

#### Caveats

- It is a single-author preprint over nine problems with known checker defects.
- The escalation tool told models their report went to a human reviewer. Whether the effect holds when the report goes to a launching agent instead is untested.
- An independent benchmark points the same way: in ImpossibleBench, a strict prompt telling the model to stop and explain flawed tests cut GPT-5's cheating from 92% to 1%.

### Putting it in a brief

```text
Tests: never modify, delete or narrow an existing test's assertions, expected values or
scope to make it pass. If you believe a test is wrong, stop and report it in your final
message under "Disputed tests": the test, the acceptance criterion it conflicts with,
what you tried, and what you recommend. Reporting a disputed test is a valid outcome.
```

The last sentence gives the agent a permitted way out. An agent told only "do not edit tests", facing a test it cannot pass, believes its task is impossible, and tampering rises in that condition.

### Where these controls sit in the harness

The brief is the weakest layer; the rest of the stack does the enforcing (see [Where tests sit in the harness]({{ '/sections/12-testing/#where-tests-sit-in-the-harness' | relative_url }})):

| Control | Layer | What it does |
|:--|:--|:--|
| The policy line and report format in the brief | 1, instructions | Guides the agent; measured to cut tampering, and still skippable |
| The disputed-tests report as a required part of the final message | 2, workflow | Builds the way out into the task |
| The [test-edit check]({{ '/sections/12-testing/test-edit-detection.html' | relative_url }}) on the branch | 3, script | Flags any pre-existing test the branch changed |
| The [read-only hook](#make-pre-existing-tests-read-only) | 3, hook | Refuses the edit at the moment it is attempted |
| The merge gate running the original tests | 4, validation | Refuses the merge when they fail |

## Who judges a disputed test

### The launcher as judge

Whoever launched the work holds the acceptance criteria and did not write the code, which makes it one natural judge. A person or a separate reviewer agent can also take the role. My harness uses the launcher, and in that setup a run works like this:

1. The agent finishes its branch and lists any disputed tests in its final message.
   - Each entry names the test, the criterion it conflicts with, what was tried and a recommendation.
2. The test-edit check lists the pre-existing tests the branch changed.
   - Changes with a stated reason are listed apart from changes without one.
3. The launcher reads those two lists against the criteria and rules on each item.
   - A branch with no dispute and no unreasoned edit goes on to normal review.

#### Worked example: the report and the ruling

```text
Disputed tests
- test_export_includes_archived_items expects 12 rows; the change exports 9.
  Conflicts with: AC3, "archived items are omitted from the default export".
  Tried: confirmed the 3 missing rows are the archived fixtures.
  Recommend: change the test to call the export with include_archived=True.
```

The launcher checks AC3, agrees, and replies with one line. The agent updates the test and records why in the commit:

```text
Test-edit-reason: AC3 changed (archived items omitted from the default export)
```

If the launcher disagrees, the reply names what the criterion actually requires, and the agent fixes the code instead.

### An LLM judge on every diff

A model asked to judge every change is costly and only partly reliable:

| Evidence | Finding |
|:--|:--|
| ImpossibleBench (2025 models) | LLM monitors caught 86% to 89% of cheating on small coding tasks and 42% to 50% on realistic repository tasks |
| Same benchmark | A "backward compatibility" rationale in the diff deceived a strong model acting as the monitor |
| EvilGenie | An LLM judge was highly effective in unambiguous cases |
| Anthropic's guidance for Claude Opus 5 | Remove routine "use a subagent to verify" lines; the model checks its own work and the extra pass wastes tokens |

The research behind this page concluded that a judge is useful on a flagged diff and costly on every diff. A setup that uses both:

1. The deterministic test-edit check runs on every branch.
2. Only a branch it flags, or one with a disputed test, goes to a judge or the launcher.

## Heavier controls

### Make pre-existing tests read-only

ImpossibleBench found read-only test files a middle ground:

- They "restore legitimate performance while preventing test modification attempts".
- They do not stop special-casing.

In Claude Code, a `PreToolUse` hook can enforce it.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "${CLAUDE_PROJECT_DIR}/tools/protect-tests.sh" }]
      }
    ]
  }
}
```

```bash
#!/usr/bin/env bash
# tools/protect-tests.sh: block edits to existing test files; new test files are allowed
path=$(jq -r '.tool_input.file_path // empty')
case "$path" in
  */tests/*|*_test.py|*/test_*.py)
    if [ -e "$path" ]; then
      echo "Existing tests are read-only in this task. Report a disputed test instead." >&2
      exit 2
    fi ;;
esac
exit 0
```

#### Costs and gaps of the hard block

| Cost or gap | Detail | Mitigation |
|:--|:--|:--|
| Blocks legitimate edits | A changed acceptance criterion, or a test that must move to a new boundary | Turn it off for tasks whose criteria change tests |
| The Bash tool is not covered | `sed -i`, `echo >` or `git checkout` can still rewrite a test file | Add a `Bash` matcher that refuses commands naming test paths, or rely on the test-edit check |
| Binds one tool only | A Claude Code hook does not bind a Codex worker or a person | The test-edit check runs on the branch, whoever wrote it |
| Misses special-casing | The code can still detect the fixture | Held-out tests the agent never sees |

It fits long unattended runs where nobody reads the test diff.

### File formats the model rewrites less

Anthropic's long-running harness kept its feature list in a JSON file that agents could change only by flipping a `passes` field, because "the model is less likely to inappropriately change or overwrite JSON files compared to Markdown files."

```json
{ "category": "functional", "description": "CSV export keeps titles with commas", "steps": ["..."], "passes": false }
```

<details markdown="1">
<summary>Aside: exploratory approaches, not yet worth adopting</summary>

- Consort (2026 preprint) argues for controls the agent runs inside but cannot bypass, including immutable tests, framed as "a pre-registered, testable hypothesis".
- SpecGuard (2026 preprint) formally checks for conflicts between spec and tests in Lean before the agent starts, and reports detecting up to 72.8% of them.
- Neither has adoption evidence yet.

</details>

## Pitfalls

| Pitfall | Why it bites | Fix |
|:--|:--|:--|
| "Do not edit tests" with no way out | The agent sees an impossible task, the condition where tampering rises | Add the disputed-tests report and say it is a valid outcome |
| Trusting a plausible comment on a changed assertion | The comment is the agent's own justification | Check the assertion against the criterion, not the comment |
| A read-only hook on Edit and Write only | Bash can still change the file | Cover Bash, or check the branch diff |
| An LLM judge on every diff | Costly, and weakest on realistic repository tasks | Judge only flagged branches |

## From my harness

My testing rule carries this as a short "oracle integrity" paragraph:

1. The agent changing the code does not weaken, delete or rewrite a pre-existing test to make its change pass.
2. If it believes the test is wrong, it stops and reports to its launcher with the test, the criterion it conflicts with, what it tried and its recommendation.
3. A pre-existing test may be edited in the same change only with a stated reason, written as a commit trailer.
   - The two accepted reasons: a changed acceptance criterion, which the trailer cites, or a test coupled to the implementation that is being moved to its real boundary.
4. Before merging a delegated branch, the launcher runs a small tool that lists every pre-existing test the branch changed and whether each change carries a reason.
   - An edit with no reason is a review finding.

```text
Test-edit-reason: acceptance criterion 3 changed (export now omits archived items by default)
```

The [sub-topic page]({{ '/sections/12-testing/test-edit-detection.html' | relative_url }}) shows how such a tool works. I have not turned on the read-only hook by default, because many of my tasks legitimately change tests.

## Sources

- [Escalation channels and reward hacking](https://arxiv.org/abs/2608.29460), verified 2026-11-09.
- [ImpossibleBench](https://arxiv.org/abs/2510.20270), verified 2026-10-08.
- [EvilGenie reward-hacking benchmark](https://arxiv.org/abs/2511.21654), verified 2026-10-08.
- [Test overfitting on SWE-bench](https://arxiv.org/abs/2511.16858), verified 2026-10-08.
- [Claude Opus 5.5 system card (impossible tasks, grader awareness)](https://www.anthropic.com/claude-opus-5-5-system-card), verified 2026-10-08.
- [Prompting Claude Opus 5 (verification instructions)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5), verified 2026-10-08.
- [Claude Code hooks reference](https://code.claude.com/docs/en/hooks), verified 2026-11-09.
- [Claude Code hooks guide: block edits to protected files](https://code.claude.com/docs/en/hooks-guide), verified 2026-10-08.
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents), verified 2026-10-08.

{% include page_nav.html %}
