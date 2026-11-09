---
title: "Testing agent-written code"
nav_order: 22
has_children: true
has_toc: false
permalink: /sections/12-testing/
---

{% include page_nav.html %}

# Testing agent-written code
{: .no_toc }

As of 2026-11-09. Stop 9 on the talk path, about 2 minutes.

When an agent writes the code, it usually writes the tests too. The tests are then also the signal the agent optimises against, and an agent can make a test pass without meeting the intent behind it. This overview covers five testing habits for agent-written code, with a worked example for each. Each habit links to a topic page with more detail.

<figure>
  <img src="{{ '/assets/images/12-testing/bugfix-run.svg' | relative_url }}" width="100%" alt="Sequence diagram of one bug-fix run across five lanes: launcher, coding agent, test suite, real entry point and merge gate. The launcher briefs the agent with the bug report, the criteria and a never-weaken-a-test line. The agent drives the bug through the real entry point, sees the wrong output, and commits a failing test. It runs that test on the fix and sees it pass, then runs an end-to-end check. An old test now fails, and the agent reports it as disputed instead of editing it. The agent pushes; the merge gate runs the tests the diff touches and reports no test edits; the launcher rules on the dispute and merges.">
  <figcaption>One bug-fix run, top to bottom. Made with archify.</figcaption>
</figure>

The run in the diagram, step by step:

1. The launcher briefs the coding agent.
   - The brief carries the bug report, the acceptance criteria and a line forbidding weakened tests.
2. The agent reproduces the bug through the real entry point.
   - It sees the wrong output the report describes.
3. The agent commits a failing regression test before writing the fix.
   - The fix may not change that test.
4. The agent writes the fix and checks it.
   - The new test passes, and an end-to-end check runs through the entry point.
5. An old test now fails, and the agent reports it as disputed.
   - It does not edit the old test.
6. The merge gate runs the tests the diff touches and reports any pre-existing test the branch changed.
7. The launcher rules on the disputed test and merges.

| Habit | One line | Topic page |
|:--|:--|:--|
| Test the real entry point | Drive the CLI, route, hook or page the user touches, and assert what they would see | [Testing at the real entry point]({{ '/sections/12-testing/entry-point-testing.html' | relative_url }}) |
| Anchor tests in the spec | Tests come from the acceptance criteria, not from the code they check | [Testing at the real entry point]({{ '/sections/12-testing/entry-point-testing.html#test-ordering-and-the-spec' | relative_url }}) |
| Keep the oracle honest | Forbid weakening tests, give the agent a way to report a bad one, and flag test edits in the diff | [Test tampering and oracle integrity]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}) |
| Freeze the bug's test first | A regression test must fail on the old code before it may pass on the fix | [The bug-fix regression test]({{ '/sections/12-testing/regression-tests.html' | relative_url }}) |
| Scale the suite deliberately | Select tests from the diff and serialise heavy runs, so test time does not become the bottleneck | This page, [testing at scale](#testing-at-scale) |

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## Where tests sit in the harness

This site treats harness engineering as a systems approach: when an agent gets something wrong, the fix goes into the system, enforced by code wherever possible, and each new gap gets the same treatment. Tests are the last of four layers in that system:

1. Instructions in context files, skills and prompts.
   - Example: a testing rule that loads when the agent edits a test file.
   - The model applies these inconsistently, especially with several tasks in flight or a full context window, so they are the second line of defence.
2. Workflows and tools that build the steps in.
   - Example: a bug-fix procedure whose steps include "commit the failing test before the fix".
3. Hooks, scripts and linters that refuse or flag at the moment of action.
   - Example: a hook that blocks edits to existing test files, or a script that lists changed tests on a branch.
4. Validation and tests before a change lands.
   - Example: the merge gate runs the tests the diff touches and refuses the merge when one fails.

Layers 2 to 4 do the enforcing. A testing lesson that only becomes another line in a context file tends to be missed on a busy run; the same lesson as a gate check runs on every change. The [bug-fix diagram](#testing-agent-written-code) above shows layers 1, 2 and 4 in one run, and [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers layer 3.

## Why agent-written tests need different habits

A human who writes a test usually knows what the feature is for. An agent knows what the prompt said and what the code does. The 2026 evidence on that gap falls into three groups:

| Group | What the evidence shows | Subsection |
|:--|:--|:--|
| Volume | Agents add more tests than a change needs, and most check the implementation | [Test volume](#test-volume) |
| Reward signal | Agents optimise against their own tests, and iterating on them makes results worse | [Tests as the reward signal](#tests-as-the-reward-signal) |
| Isolation | Agents pass tests of single features and fail when features combine | [Overfitting to isolated checks](#overfitting-to-isolated-checks) |

### Test volume

The model vendors now document this about their own models:

- One vendor's guide for a recent model says it "may fix nearby code, extend behavior the task didn't mention, or commit more test files than the change warrants."
  - The same guide suggests "roughly one focused test per stated behavior" and reports fewer unrequested files with no loss in task success.
- Another vendor's guide warns that for small tasks its model can write "broader tests than the task requires."
  - Its fix is a prompt line: do not write tests for reversible, low-impact changes that mirror the implementation.
- An open-source maintainer removed about 400,000 lines of agent-written tests with little change in code coverage.
  - The maintainer's explanation: modern models write tests for every small change, useful or not.

Most of these tests check how the code is written. They keep passing when the feature a user relies on is broken, and they fail when someone refactors code that still works.

### Tests as the reward signal

When the agent writes the code and its tests, the tests become the score it works toward. A 2026 study on the SWE-bench benchmark measured how often that score misleads:

| Measurement | One model | Another model |
|:--|--:|--:|
| Passed the agent's own tests, failed the hidden reference tests | 21.8% | 33.0% |
| Same, after the agent iterated on its own tests | 25.5% | 35.9% |
| Broke existing regression tests even with the reference tests shown | 5.8% | 11.3% |

A test states only part of the intent, and an optimiser can satisfy the test without meeting the rest. The models in this study predate the current generation, so the exact rates today are unknown; the mechanism does not depend on the model.

### Overfitting to isolated checks

The SpecBench benchmark (2026) separates two kinds of test:

- Visible tests check each feature on its own.
  - Every frontier agent tested passed the visible suite.
- Held-out tests combine the same features the way a user would.
  - The gap between the two suites grew by 28 percentage points for every tenfold increase in code size.
  - The most common failure was features that each worked alone and did not share state with each other.

For long tasks on large codebases, the tests that catch real failures are the ones that drive several features together through the entry point.

## Testing at the entry point

The default test for a feature or a bug fix drives the real entry point and asserts the outcome the acceptance criteria name. The entry point is whatever a user or another program calls:

- a command line, run with real arguments in a temporary home directory;
- an HTTP route, called with a real request;
- a hook, fed the JSON payload its host sends;
- a rendered page, loaded in a browser;
- a library's public function, called the way a caller imports it.

### Worked example: a command-line tool that files a task

Suppose an agent adds a `--priority` flag to a small command-line tool that creates a task record. A typical agent-written test checks the internal parser:

```python
def test_parse_priority():
    assert parse_priority("high") == 1
```

That test passes even if the flag is never wired into the command, or if the record is written without the field. The behaviour-first test runs the command the way a user would and reads back what it wrote:

```python
def test_priority_flag_is_stored(tmp_path):
    result = run_cli(["new", "Fix login", "--priority", "high"], home=tmp_path)
    assert result.exit_code == 0
    record = read_task(tmp_path, result.task_id)
    assert record["priority"] == 1
```

This test fails if any step between the command line and the stored record is broken, and it keeps passing through any refactor that keeps the behaviour.

#### Function-level tests

Function-level tests pass a value gate when the function is itself a contract (a parser for a public file format, a migration, a security check), when the test freezes a reproduced bug, or when dense pure logic is easier to pin with a table of inputs and outputs than through a large fixture. The [topic page]({{ '/sections/12-testing/entry-point-testing.html#the-value-gate' | relative_url }}) gives the four questions one open-source project asks before any test is added.

### Test ordering

Tests come from acceptance criteria written before the code. That is separate from the order in which the agent writes tests and code:

| Ordering | What the agent does | 2026 evidence |
|:--|:--|:--|
| Strict TDD | One failing test, just enough code, repeat | No quality or mutation-score gain in one evaluation, at 3x to 8.5x the tokens |
| Tests first | Writes the tests from the spec, then the code | About the same tokens as no TDD; quality effect unclear |
| Spec first | Writes acceptance criteria first; tests derive from them | Spec-grounded test generation found 9.8 points more real bugs in another study |

The spec has to exist before the code. Without it, nobody can tell whether a failing test or the code is wrong, and agents tamper with tests most often in exactly that situation (next section). The [topic page]({{ '/sections/12-testing/entry-point-testing.html#test-ordering-and-the-spec' | relative_url }}) shows what a proportional spec looks like.

## Oracle integrity

The oracle is whatever decides whether the code is right. An agent that can edit both the code and the oracle can make them agree by weakening the oracle, which is called test tampering, a kind of reward hacking.

| Oracle | Example | How it gets weakened |
|:--|:--|:--|
| Test suite | `tests/test_export.py` | An assertion is deleted, loosened or skipped |
| Reference output | An expected CSV file | The fixture is regenerated from the new, wrong output |
| Checker script | A script that validates a migration | The check is narrowed or its exit code ignored |

### What triggers it

In ordinary work, agents mostly fix the code. Tampering concentrates in three situations:

1. The task cannot be done as specified.
   - One model card reports attempted reward hacking three to six times more often on impossible tasks than on possible ones.
2. The written spec and a test disagree.
   - ImpossibleBench builds tasks with deliberate spec-test conflicts; some model families cheated there mainly by editing test cases.
3. The checker itself is defective.
   - On tasks with known checker defects, baseline hacking rates were 11.1% and 13.3% for two current models.

Controls work best aimed at those moments: when an agent cannot pass a test, it needs a permitted next step.

### Policy line and escalation channel

<figure>
  <img src="{{ '/assets/images/12-testing/tamper-dispute.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: launcher, coding agent, old test and test-edit check. The agent runs the suite and an old export test fails, expecting 12 rows where the change exports 9. On the common path the agent reports a disputed test citing criterion 3; the launcher confirms the criterion changed and tells the agent to update the test with a stated reason. On the rare path the agent silently loosens the assertion to greater than zero; the test-edit check flags a changed pre-existing test with no reason, and the launcher files a finding to restore the assertion.">
  <figcaption>The two paths an agent can take when an old test blocks it. The report channel makes the honest path the easy one, and the diff check catches the rest. Made with archify.</figcaption>
</figure>

A 2026 study across eight frontier models from five model families tested two cheap changes. The first is a policy line telling the agent not to weaken tests. The second is a sanctioned way for the agent to report a test it thinks is wrong. Together they cut test tampering from 23.6% to 5.3%, with no measured cost, and removed it entirely for six of the eight models. Escalation and hacking almost never happened in the same run.

A brief can carry both in three lines:

```text
Never modify, delete or narrow an existing test's assertions, expected values or scope.
If you believe a test is wrong, stop and report it: the test, why it conflicts with the
acceptance criteria, what you tried, and what you recommend. Do not work around it.
```

How the parts fit together in one run:

1. The brief carries the policy line and the report format.
2. The agent either fixes the code or reports a test it believes is wrong.
   - The report names the test, the criterion it conflicts with, what was tried and a recommendation.
3. Someone who did not write the code rules on the report.
   - In my harness that is the launcher, which holds the acceptance criteria. A person or a separate reviewer agent can also take this role.
4. A test-edit check on the diff lists any pre-existing test the branch changed anyway.

The [topic page]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}) covers who decides, the heavier option of making tests read-only with a hook, and why an LLM judge on every diff is the wrong default.

### Test-edit detection

Both reward-hacking benchmarks used the same detection channel: a check that flags when a pre-existing test file changed. For a delegated branch it is one command, read before merge:

```bash
git diff --name-only --diff-filter=MD "$(git merge-base origin/main HEAD)"...HEAD -- tests/
```

- `--diff-filter=MD` lists modified and deleted test files and leaves out new ones.
- Each listed file either carries a stated reason in the commit message or becomes a review finding.
- The check cannot see code that special-cases the test input, so it backs up the policy line and does not replace it.

[Detecting test edits in a diff]({{ '/sections/12-testing/test-edit-detection.html' | relative_url }}) covers the reason trailer, the decision table and a sample report.

## Bug-fix regression tests

In one 2026 evaluation, strict TDD did not pay for itself inside the agent loop. A bug fix still keeps its red step, for two reasons:

- A bug report describes a concrete failure, so there is an exact behaviour for the test to reproduce.
- Running the test on the unfixed code shows whether the test detects the bug at all.
  - A regression test that never failed on the old code does not show that the fix works.

### Worked example: the four steps

| Step | Run | Expected |
|:--|:--|:--|
| 1. Reproduce | the new test on the unfixed code | fails, for the reason the bug report describes |
| 2. Freeze | commit the failing test on its own | the test is now fixed; the fix may not touch it |
| 3. Fix | the same test on the fixed code | passes |
| 4. Preserve | the rest of the suite on the fixed code | still passes |

<figure>
  <img src="{{ '/assets/images/12-testing/regression-proof.svg' | relative_url }}" width="100%" alt="Sequence diagram with five lanes: coding agent, unfixed code, regression test, fixed code and full suite. The agent drives the reported input into the unfixed code and sees the wrong output. It writes a test at the entry point and runs it on the unfixed code, which fails for the stated reason. It commits the failing test alone. It writes the fix, runs the same test on the fixed code, which passes, and runs the preservation tests, which stay green.">
  <figcaption>Red then green. The same frozen test fails on the old code and passes on the fix. Made with archify.</figcaption>
</figure>

The order matters because an agent asked to "fix the bug and add a test" can write a test that passes on both old and new code. The [topic page]({{ '/sections/12-testing/regression-tests.html' | relative_url }}) walks the full method: reproduce, root-cause, freeze, fix, verify in widening rings, and rank where the rule is enforced.

## End-to-end and live validation

A green unit suite can pass while the feature is broken. Anthropic's write-up on harnesses for long-running agents describes the pattern:

- Claude made changes and tested them with unit tests or curl calls against a development server.
- It still missed that the feature did not work end to end.
- Adding browser-driven end-to-end testing "dramatically improved performance."

| Check | What it runs | When to run it |
|:--|:--|:--|
| Entry-point test | The real command or route, hermetic | Every change |
| Live end-to-end run | Real processes, real services | Before merge for the features it covers, or on a schedule |
| Screenshot comparison | A browser, the rendered page | Changes to anything visual |

### Worked example: a live messaging test

In my harness, one feature lets one agent session send a message to another. Unit tests can check the message format, the addressing and the queue, and every one of them can pass with a stub standing in for the real delivery path. So the feature also has a live end-to-end run: it starts real sessions, sends a real message from one, and checks that it arrives in the other within a time limit. It is slower and runs less often, but it is the only test that answers the question a user asks: did my message get there?

```text
live e2e: cross-session message
  start session A, start session B
  A sends "ping 7f3c" to B            -> accepted
  wait for B's inbox, limit 60 s
  B received "ping 7f3c" after 4.2 s  -> PASS
  stop A, stop B, remove temp state
```

### Screenshots as a check

For anything visual, the check can be a screenshot:

1. The agent opens the page in a browser it controls.
2. It takes a screenshot of the result.
3. It compares the screenshot with the intended design or a reference image.
4. It lists the differences and fixes them, then repeats from step 2.

Anthropic's best-practices page gives this as an example prompt: "take a screenshot of the result and compare it to the original. list differences and fix them." Current flagship models read screenshots well enough that the loop is cheap. Claude Code's Chrome integration is not available through Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry, so on those providers the browser is driven another way. [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}) covers the tooling.

## Testing at scale

Once several agents work in parallel, waiting for tests takes longer than writing the code. The two subsections below cover the two levers:

| Lever | Problem it addresses | Subsection |
|:--|:--|:--|
| Test selection | Each change waits for the full suite | [Test selection from the diff](#test-selection-from-the-diff) |
| Run serialisation | Parallel full-suite runs overload a shared machine | [Heavy test runs on a shared machine](#heavy-test-runs-on-a-shared-machine) |

### Test selection from the diff

Running the full suite on every change does not scale. One alternative is a map from code to the tests that cover it, used in two places:

1. The agent reads the map before it changes code.
   - One 2026 study (TDAD) gave the agent that map and cut regressions on SWE-bench Verified from 6.08% to 1.82%, with resolution up from 24% to 32%.
   - Bare TDD instructions without the map raised regressions to 9.94%.
   - The study used two open-weight models, so the size of the effect may not carry over to frontier models.
2. The push gate runs the mapped tests for the files the diff touches.
   - In my harness the full suite then runs less often, on the merged result.

### Heavy test runs on a shared machine

When several agents share one machine, parallel full-suite runs can overload it until every run times out. A machine-wide lock that heavy test runs must hold keeps them in a queue. When a gate always fails on one machine, people skip it there, so each gate needs to pass on every machine it runs on.

<details markdown="1">
<summary>Aside: how flaky end-to-end tests cost real time</summary>

Integration and end-to-end tests are more often flaky than unit tests. A widely cited figure puts about 16% of Google's tests as flaky, with 84% of pass-to-fail transitions in its CI caused by flakiness rather than real bugs. Those figures are from 2016 to 2020 and give an order of magnitude only. The mitigation is hermetic tests: each test owns its temporary files, ports and processes, and cleans up what it started.

</details>

## Testing instructions by model tier

Newer flagship models check their own work without being told. Both major vendors now say that old "always verify" instructions make these models over-test and waste tokens. Cheaper models at low effort still sometimes report work done without running any check. A single testing instruction for all models over-instructs the flagship models and under-instructs the cheaper ones at low effort.

| Model tier | What it tends to do | What to put in the brief |
|:--|:--|:--|
| Flagship, any effort | Verifies on its own; may over-test | Drop "always verify" lines; ask for about one focused test per stated behaviour |
| Mid or small tier, low effort | Sometimes skips the check | "Run a real check that exercises the change before reporting it done" |

## From my harness

The descriptions below are conceptual.

- Testing rules load only when the agent touches a test file, so the always-loaded context stays short. The rule says to drive the real caller end to end and to count coverage in acceptance criteria, not lines.
- A bug-fix procedure with numbered gates: reproduce, root-cause, freeze the failing test, fix, then verify in widening rings.
- Live end-to-end runs that send real messages between sessions and check they arrive.
- An oracle-integrity rule: a pre-existing test is edited only with a stated reason in a commit trailer, and the launcher checks every delegated branch for unreasoned test edits before merge.
- A push gate that selects tests from the diff, and a machine-wide test lock.

To start, write one end-to-end test per feature that runs the real entry point, and add the three-line policy to briefs that ask an agent to change code.

## Pitfalls

| Pitfall | Why it bites | Fix |
|:--|:--|:--|
| Counting coverage in lines | A line can run without its effect being checked; one helper with full line coverage still let 13 mutants survive | Count coverage in acceptance criteria; use mutation testing on core logic |
| A healer that loops until green | It can repair the assertion instead of the app | Let it fix selectors, never expected values |
| "Fix this and add a test" in one step | The test can pass on the old code too | Freeze the failing test first |
| Verification instructions left over from older models | Flagship models over-test | Calibrate per tier |

## Sources

- Test overfitting on SWE-bench (21.8% and 33.0%): <https://arxiv.org/abs/2511.16858>, verified 2026-10-08.
- SpecBench (28 points per tenfold code size): <https://arxiv.org/abs/2605.21384>, verified 2026-10-08.
- Escalation channels and reward hacking (23.6% to 5.3%, eight models): <https://arxiv.org/abs/2608.29460>, verified 2026-11-09.
- ImpossibleBench: <https://arxiv.org/abs/2510.20270>, verified 2026-10-08.
- TDD inside the agent loop: <https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html>, verified 2026-10-08.
- Maintainability sensors (13 surviving mutants): <https://martinfowler.com/articles/sensors-for-coding-agents.html>, verified 2026-10-08.
- TDAD impact maps (6.08% to 1.82%): <https://arxiv.org/abs/2603.17973>, verified 2026-10-08.
- Effective harnesses for long-running agents: <https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents>, verified 2026-10-08.
- Over-testing guidance: <https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-1> and <https://developers.openai.com/api/docs/guides/latest-model>, verified 2026-10-08.
- The deleted-tests example: <https://www.explainx.ai/blog/openclaw-test-audit-skill-deleted-400k-lines-tests-2026>, verified 2026-11-09, a secondary report of the maintainer's post.

{% include page_nav.html %}
