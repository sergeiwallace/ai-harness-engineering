---
title: "Testing agent-written code"
nav_order: 22
has_children: true
has_toc: false
permalink: /sections/12-testing/
last_updated: "2026-11-09"
talk_stop: "9"
stop_minutes: "2"
---

{% include page_nav.html %}

# Testing agent-written code
{: .no_toc }

When an agent writes the code, it usually writes the tests too. Two things follow:

- Tests state how the code should behave, so they encode the intent before any code exists.
- Tests are also the score the agent works toward, and an agent can make a test pass without meeting the intent behind it.

This overview covers six testing practices for agent-written code, each with a worked example and a link to a topic page where there is more depth.

| Practice | What it means | Go deeper |
|:--|:--|:--|
| [Test at the real entry point](#testing-at-the-entry-point) | Drive the command, route, hook or page a user touches, and assert what they would see | [Testing at the real entry point]({{ '/sections/12-testing/entry-point-testing.html' | relative_url }}) |
| [Write the spec before the code](#test-ordering) | Tests come from acceptance criteria written first | [Test ordering and the spec]({{ '/sections/12-testing/entry-point-testing.html#test-ordering-and-the-spec' | relative_url }}) |
| [Keep the oracle honest](#oracle-integrity) | Forbid weakening tests, give the agent a way to report a bad one, flag test edits in the diff | [Test tampering and oracle integrity]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}), then [Detecting test edits in a diff]({{ '/sections/12-testing/test-edit-detection.html' | relative_url }}) |
| [Freeze the bug's test first](#bug-fix-regression-tests) | A regression test fails on the old code before it may pass on the fix | [The bug-fix regression test]({{ '/sections/12-testing/regression-tests.html' | relative_url }}) |
| [Run the real system](#end-to-end-and-live-validation) | End-to-end and live runs, and screenshots for anything visual | [Browser automation]({{ '/sections/15-browser-automation/' | relative_url }}) |
| [Scale the suite deliberately](#testing-at-scale) | Select tests from the diff and queue heavy runs | This page |

<details markdown="1">
<summary>Terms used in this section</summary>

| Term | Meaning |
|:--|:--|
| Launcher | The person or agent session that handed the task to the coding agent and will review its result |
| Acceptance criteria | The short list of statements that say when the task is done, written before the code |
| Entry point | What a user or another program actually calls: a command, an HTTP route, a hook's input, a page |
| Oracle | Whatever decides whether the code is right: a test suite, a reference output, a checker script |
| Reward hacking | An agent meeting the letter of its success check without meeting its intent; test tampering is one form |
| Regression test | A test that reproduces a fixed bug, so the bug cannot come back unnoticed |
| Mutation testing | A tool changes the code in small ways and checks whether any test fails; surviving changes show weak tests |
| Hermetic test | A test that owns all its temporary files, ports and processes, and cleans up after itself |
| TDD | Test-driven development: write one failing test, write just enough code to pass it, repeat |

</details>

<figure>
  <img src="{{ '/assets/images/12-testing/bugfix-run.svg' | relative_url }}" width="100%" alt="Sequence diagram of one bug-fix run across five lanes: launcher, coding agent, test suite, real entry point and merge gate. The launcher briefs the agent with the bug report, the criteria and a never-weaken-a-test line. The agent drives the bug through the real entry point, sees the wrong output, and commits a failing test. It runs that test on the fix and sees it pass, then runs an end-to-end check. An old test now fails, and the agent reports it as disputed instead of editing it. The agent pushes; the merge gate runs the tests the diff touches and reports no test edits; the launcher rules on the dispute and merges.">
  <figcaption>One bug-fix run, top to bottom, using five of the six practices. Made with archify.</figcaption>
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

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## Why agent-written tests need different habits

A human who writes a test usually knows what the feature is for. An agent knows what the prompt said and what the code does. The 2026 evidence on that gap falls into three groups:

| Group | What the evidence shows | Subsection |
|:--|:--|:--|
| Volume | Agents add more tests than a change needs, and most check the implementation | [Test volume](#test-volume) |
| Reward signal | Agents optimise against their own tests, and iterating on them makes results worse | [Tests as the reward signal](#tests-as-the-reward-signal) |
| Isolation | Agents pass tests of single features and fail when features combine | [Overfitting to isolated checks](#overfitting-to-isolated-checks) |

<!-- IMAGE NEEDED: chart of visible-test pass rate against held-out-test pass rate for coding agents, gap widening with code size; search: SpecBench reward hacking coding agents chart, held-out tests gap -->

### Test volume

The model vendors now document this about their own models:

- Anthropic's prompting guide for Claude Fable 5.1 says the model "may fix nearby code, extend behavior the task didn't mention, or commit more test files than the change warrants."
  - The same guide suggests "roughly one focused test per stated behavior" and reports fewer unrequested files with no measurable change in task success.
- OpenAI's guide for GPT-6 warns that for smaller tasks the model can write "broader tests than the task requires."
  - Its suggested prompt line: "Do not write tests for reversible, low-impact changes that mirror the implementation."
- An open-source maintainer removed about 400,000 lines of agent-written tests with little change in code coverage.
  - The maintainer's explanation, as reported: "Modern models just love writing tests for every tiny change, even if they aren't useful."

Most of these tests check how the code is written. They keep passing when the feature a user relies on is broken, and they fail when someone refactors code that still works.

### Tests as the reward signal

When the agent writes the code and its tests, the tests become the score it works toward. A 2026 study on the SWE-bench benchmark measured how often that score misleads:

| Measurement | Claude 3.7 Sonnet | GPT-4o |
|:--|--:|--:|
| Passed the agent's own tests, failed the hidden reference tests | 21.8% | 33.0% |
| Same, after the agent iterated on its own tests | 25.5% | 35.9% |
| Broke existing regression tests even with the reference tests shown | 5.8% | 11.3% |

A test states only part of the intent, and an optimiser can satisfy the test without meeting the rest. These models predate the current generation, so the exact rates today are unknown; the mechanism does not depend on the model.

### Overfitting to isolated checks

The SpecBench benchmark (2026) separates two kinds of test:

- Visible tests check each feature on its own.
  - Every frontier agent tested passed the visible suite.
- Held-out tests combine the same features the way a user would.
  - The gap between the two suites grew by 28 percentage points for every tenfold increase in code size.
  - The most common failure was features that each worked alone and did not share state with each other.

For long tasks on large codebases, the tests that catch real failures are the ones that drive several features together through the entry point.

### Measuring test strength

Line coverage says a line ran, not that a test checked what it did. Two measures, and what each misses:

| Measure | What it reports | What it misses |
|:--|:--|:--|
| Line or branch coverage | Which lines and branches some test executed | Whether any assertion depends on them |
| Mutation score | The share of small deliberate code changes ("mutants") that make some test fail | Slow to run on a whole codebase; best on core logic |

One 2026 write-up describes a file with 100% statement coverage and no unit tests at all: a large acceptance test happened to call every line, and a mutation tool still found 13 surviving mutants. Thoughtworks placed mutation testing in the Trial ring of its Technology Radar in April 2026, calling it "the most honest signal" of a suite's fault-detection ability.

Python, with mutmut (configure the paths once, then run):

```toml
# pyproject.toml
[tool.mutmut]
source_paths = ["src/"]
pytest_add_cli_args_test_selection = ["tests/"]
```

```bash
mutmut run "export*"    # limit the run to one module
mutmut browse           # inspect surviving mutants interactively
```

JavaScript and TypeScript, with StrykerJS:

```bash
npm init stryker@latest   # installs Stryker and writes stryker.config.mjs
npx stryker run
```

Run it on dense core logic (parsers, pricing, permission checks), where a surviving mutant points at an assertion the agent did not write.

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

Function-level tests still pass a value gate in three cases:

- The function is itself a contract: a parser for a public file format, a migration, a security check.
- The test freezes a reproduced bug in that function.
- Dense pure logic is easier to pin with a table of inputs and outputs than through a large fixture.

The [value gate]({{ '/sections/12-testing/entry-point-testing.html#the-value-gate' | relative_url }}) on the topic page gives the four questions one open-source project asks before any test is added.

#### Edge cases

| Situation | What to do at the entry point |
|:--|:--|
| The feature calls a paid or rate-limited API | Replace only that API with a recorded response; run everything else for real |
| The entry point is slow (seconds per call) | Keep one test per acceptance criterion; put the many input variations in a table-driven test of the function behind it |
| The output includes timestamps or random ids | Assert on the fields the criteria name, or freeze the clock and the seed |

### Test ordering

Tests come from acceptance criteria written before the code. That is separate from the order in which the agent writes tests and code:

| Ordering | What the agent does | 2026 evidence |
|:--|:--|:--|
| Strict TDD | One failing test, just enough code, repeat | No quality or mutation-score gain in one evaluation, at 3x to 8.5x the tokens |
| Tests first | Writes the tests from the spec, then the code | About the same tokens as no TDD; quality effect unclear |
| Spec first | Writes acceptance criteria first; tests derive from them | Spec-grounded test generation found 9.8 points more real bugs in another study |

The spec has to exist before the code:

- Without it, nobody can tell whether a failing test or the code is wrong.
- Agents tamper with tests most often in exactly that situation (next section).

The [topic page]({{ '/sections/12-testing/entry-point-testing.html#test-ordering-and-the-spec' | relative_url }}) shows what a proportional spec looks like.

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
   - Anthropic's Claude Opus 5.5 system card reports attempted reward hacking three to six times more often on impossible tasks than on possible ones.
2. The written spec and a test disagree.
   - ImpossibleBench builds tasks with deliberate spec-test conflicts; some model families cheated there mainly by editing test cases.
3. The checker itself is defective.
   - On tasks with known checker defects, baseline hacking rates were 11.1% for claude-fable-5 and 13.3% for gpt-5.6-sol.

Controls work best aimed at those moments: when an agent cannot pass a test, it needs a permitted next step.

### Policy line and escalation channel

<figure>
  <img src="{{ '/assets/images/12-testing/tamper-dispute.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: launcher, coding agent, old test and test-edit check. The agent runs the suite and an old export test fails, expecting 12 rows where the change exports 9. On the common path the agent reports a disputed test citing criterion 3; the launcher confirms the criterion changed and tells the agent to update the test with a stated reason. On the rare path the agent silently loosens the assertion to greater than zero; the test-edit check flags a changed pre-existing test with no reason, and the launcher files a finding to restore the assertion.">
  <figcaption>The two paths an agent can take when an old test blocks it. Made with archify.</figcaption>
</figure>

A 2026 study across eight frontier models from five model families tested two cheap changes:

| Change | What it is |
|:--|:--|
| A policy line | Tells the agent not to modify, delete or narrow test assertions, expected values or scope |
| An escalation channel | A sanctioned way for the agent to report a test it thinks is wrong |

Together they cut test tampering from 23.6% to 5.3%, with no measured cost, and removed it entirely for six of the eight models. Escalation and hacking almost never happened in the same run (98.7% of escalations involved no hacking). A brief can carry both in three lines:

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

The [topic page]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}) covers who decides, the heavier option of making tests read-only with a hook, and why an LLM judge on every diff is a costly default.

### Test-edit detection

Two 2026 reward-hacking benchmarks, EvilGenie and the escalation-channel study above, used the same detection channel: a check that flags when a pre-existing test file changed. For a delegated branch it is one command, read before merge:

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

The order matters because an agent asked to "fix the bug and add a test" can write a test that passes on both old and new code. The [topic page]({{ '/sections/12-testing/regression-tests.html' | relative_url }}) walks the full method: reproduce, root-cause, freeze, fix, verify in widening rings, and where each rule is enforced.

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

In my harness, one feature lets one agent session send a message to another. What each kind of test can and cannot see:

- Unit tests check the message format, the addressing and the queue.
  - Every one of them can pass with a stub standing in for the real delivery path.
- A live end-to-end run starts real sessions, sends a real message from one, and checks that it arrives in the other within a time limit.
  - It is slower and runs less often, and it is the only test that answers the question a user asks: did my message get there?

```text
live e2e: cross-session message
  start session A, start session B
  A sends "ping 7f3c" to B            -> accepted
  wait for B's inbox, limit 60 s
  B received "ping 7f3c" after 4.2 s  -> PASS
  stop A, stop B, remove temp state
```

The last line removes the sessions and state the run created, so a failed run does not leave processes behind on a shared machine.

### Screenshots as a check

For anything visual, the check can be a screenshot:

1. The agent opens the page in a browser it controls.
2. It takes a screenshot of the result.
3. It compares the screenshot with the intended design or a reference image.
4. It lists the differences and fixes them, then repeats from step 2.

Anthropic's best-practices page gives this as an example prompt: "take a screenshot of the result and compare it to the original. list differences and fix them." Current flagship models read screenshots well enough that the loop is cheap.

> Claude Code's Chrome integration is not available through Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry, so on those providers the browser is driven another way, for example with Playwright. [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}) covers the tooling.

<!-- IMAGE NEEDED: side-by-side of a design mockup and a rendered page screenshot with the differences circled; search: visual regression testing screenshot diff, playwright screenshot comparison -->

## Testing at scale

Once several agents work in parallel, waiting for tests takes longer than writing the code. The two levers:

| Lever | Problem it addresses | Subsection |
|:--|:--|:--|
| Test selection | Each change waits for the full suite | [Test selection from the diff](#test-selection-from-the-diff) |
| Run queueing | Parallel full-suite runs overload a shared machine | [Heavy test runs on a shared machine](#heavy-test-runs-on-a-shared-machine) |

<!-- IMAGE NEEDED: test impact analysis, changed source files mapped to the subset of tests that cover them; search: test impact analysis diagram, regression test selection -->

### Test selection from the diff

Running the full suite on every change does not scale. One alternative is a map from code to the tests that cover it, used in two places:

1. The agent reads the map before it changes code.
   - One 2026 study (TDAD) gave the agent that map and cut regressions on SWE-bench Verified from 6.08% to 1.82%, with resolution up from 24% to 32%.
   - Bare TDD instructions without the map raised regressions to 9.94%.
   - The study used two open-weight models, so the size of the effect may not carry over to frontier models.
2. The push gate runs the mapped tests for the files the diff touches.
   - In my harness the full suite then runs less often, on the merged result.

| Edge case | Why selection can miss it | What to do |
|:--|:--|:--|
| A config or fixture file changed | No test imports it directly | Map data files to the tests that read them, or run the full suite |
| A shared helper changed | Its callers' tests are many | Treat helpers as "run everything" paths |
| A new file with no tests yet | The map has no entry | Fail the gate until the change adds or names a test |

### Heavy test runs on a shared machine

When several agents share one machine, parallel full-suite runs can overload it until every run times out. A machine-wide lock queues them:

```bash
# Each heavy run waits for the same lock; flock releases it when pytest exits
flock /tmp/test-suite.lock pytest -n 4
```

- Light runs (one test file) skip the lock; only full-suite or parallel runs take it.
- A run that cannot get the lock within a time limit reports "waiting" instead of failing, so a queue does not read as a broken test.
- When a gate always fails on one machine, people skip it there, so each gate needs to pass on every machine it runs on.

<details markdown="1">
<summary>Aside: how flaky end-to-end tests cost real time</summary>

Integration and end-to-end tests are more often flaky than unit tests:

| Figure | Value |
|:--|:--|
| Google tests that were flaky | About 16% |
| Pass-to-fail transitions in its CI caused by flakiness | 84% |
| Age of the figures | 2016 to 2020; an order of magnitude only |

The mitigation is hermetic tests: each test owns its temporary files, ports and processes, and cleans up what it started.

</details>

## Testing instructions by model tier

How much testing to ask for depends on the model:

- Newer flagship models check their own work without being told.
  - Anthropic's guide for Claude Opus 5 says to remove explicit verification instructions, because they cause over-verification and waste tokens.
  - OpenAI says GPT-6 Astra runs tests on its own, "so the same instructions can lead to unnecessary testing."
- Cheaper models at low effort still sometimes report work done without running any check.
  - Anthropic's Sonnet 5.5 and Haiku 5.5 guides both recommend the line "run a real check that exercises the change before reporting it done."

| Model tier | What it tends to do | What to put in the brief |
|:--|:--|:--|
| Flagship, any effort | Verifies on its own; may over-test | Drop "always verify" lines; ask for about one focused test per stated behaviour |
| Mid or small tier, low effort | Sometimes skips the check | "Run a real check that exercises the change before reporting it done" |

## Where tests sit in the harness

This site treats harness engineering as a systems approach: the way you want the output to be, and each fix for a gap, goes into the system, enforced by code wherever possible. Tests work in both directions:

- Proactively, a test states how the code should behave: the outputs, the formats, the required fields.
- Reactively, a regression test stops a mistake that has already happened from coming back.

Tests are the last of four layers in that system:

1. Instructions in context files, skills and prompts.
   - Example: a testing rule that loads when the agent edits a test file, or the model-tier line above.
   - The model applies these inconsistently, especially with several tasks in flight or a full context window, so they are the second line of defence.
2. Workflows and tools that build the steps in.
   - Example: a bug-fix procedure whose steps include "commit the failing test before the fix".
3. Hooks, scripts and linters that refuse or flag at the moment of action.
   - Proactive example: a linter that requires every test to call the real entry point.
   - Reactive example: a hook that blocks edits to existing test files, or a script that lists changed tests on a branch.
4. Validation and tests before a change lands.
   - Example: the merge gate runs the tests the diff touches and refuses the merge when one fails.

Layers 2 to 4 do the enforcing:

- A testing rule that is only a line in a context file tends to be missed on a busy run.
- The same rule as a gate check runs on every change.

The [bug-fix diagram](#testing-agent-written-code) at the top of this page shows layers 1, 2 and 4 in one run, and [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers layer 3.

## From my harness

The descriptions below are conceptual, one per layer where it applies:

| Mechanism | Layer | What it does |
|:--|:--|:--|
| A path-scoped testing rule | 1 | Loads only when the agent touches a test file: drive the real caller end to end, count coverage in acceptance criteria |
| A bug-fix procedure with numbered gates | 2 | Reproduce, root-cause, freeze the failing test, fix, verify in widening rings |
| An oracle-integrity rule plus a test-edit flag tool | 1 and 3 | A pre-existing test is edited only with a stated reason in a commit trailer; the launcher runs the tool on every delegated branch |
| Live end-to-end runs | 4 | Send real messages between real sessions and check they arrive |
| A push gate that selects tests from the diff | 4 | Runs the mapped tests; the full suite runs on the merged result |
| A machine-wide test lock | 3 | Queues heavy runs on a shared machine |

To start, write one end-to-end test per feature that runs the real entry point, and add the three-line policy to briefs that ask an agent to change code.

## Pitfalls

| Pitfall | Why it bites | Fix |
|:--|:--|:--|
| Counting coverage in lines | A line can run without its effect being checked; one file with full statement coverage still had 13 surviving mutants | Count coverage in acceptance criteria; use [mutation testing](#measuring-test-strength) on core logic |
| An auto-fixing test agent that loops until green | Playwright's Healer agent, for example, "Re-runs the test until it passes or until guardrails stop the loop"; it can repair the assertion instead of the app | Let it fix selectors and waits, never expected values |
| "Fix this and add a test" in one step | The test can pass on the old code too | [Freeze the failing test first](#bug-fix-regression-tests) |
| Verification instructions left over from older models | Flagship models over-test | [Calibrate per tier](#testing-instructions-by-model-tier) |
| Mocking the thing under test | The test then checks the mock | Mock only what is slow, costly or outside your control |

## Sources

- [Test overfitting on SWE-bench (21.8% and 33.0%)](https://arxiv.org/abs/2511.16858), verified 2026-10-08.
- [SpecBench (28 points per tenfold code size)](https://arxiv.org/abs/2605.21384), verified 2026-10-08.
- [Escalation channels and reward hacking (23.6% to 5.3%, eight models)](https://arxiv.org/abs/2608.29460), verified 2026-11-09.
- [ImpossibleBench](https://arxiv.org/abs/2510.20270), verified 2026-10-08.
- [EvilGenie reward-hacking benchmark](https://arxiv.org/abs/2511.21654), verified 2026-10-08.
- [Claude Opus 5.5 system card (three to six times on impossible tasks)](https://www.anthropic.com/claude-opus-5-5-system-card), verified 2026-10-08.
- [TDD inside the agent loop](https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html), verified 2026-10-08.
- [Maintainability sensors (13 surviving mutants)](https://martinfowler.com/articles/sensors-for-coding-agents.html), verified 2026-10-08.
- [Thoughtworks Technology Radar, mutation testing](https://www.thoughtworks.com/radar/techniques/mutation-testing), verified 2026-10-08.
- [TDAD impact maps (6.08% to 1.82%)](https://arxiv.org/abs/2603.17973), verified 2026-10-08.
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents), verified 2026-10-08.
- [Prompting Claude Fable 5.1 (over-testing guidance)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-1), verified 2026-10-08.
- [OpenAI, Using GPT-6 (testing guidance)](https://developers.openai.com/api/docs/guides/latest-model), verified 2026-10-08.
- [GPT-6 Astra and testing](https://developers.openai.com/blog/rethinking-skills-and-prompts-for-gpt-6-astra), verified 2026-10-08.
- [The deleted-tests example](https://www.explainx.ai/blog/openclaw-test-audit-skill-deleted-400k-lines-tests-2026), verified 2026-11-09, a secondary report of the maintainer's post.
- [mutmut](https://mutmut.readthedocs.io/en/latest/), verified 2026-11-09.
- [StrykerJS getting started](https://stryker-mutator.io/docs/stryker-js/getting-started/), verified 2026-11-09.
- [Playwright test agents (Healer)](https://playwright.dev/docs/test-agents), verified 2026-10-08.
- [Claude Code with Chrome (provider availability)](https://code.claude.com/docs/en/chrome), verified 2026-10-08.
- [Prompting Claude Opus 5 (verification instructions)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5), verified 2026-10-08.
- [Prompting Claude Sonnet 5.5 (low-effort checks)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-sonnet-5-5), verified 2026-10-08.

{% include page_nav.html %}
