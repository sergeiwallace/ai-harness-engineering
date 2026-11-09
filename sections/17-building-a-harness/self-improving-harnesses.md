---
title: "Self-improving harness research"
parent: "The improvement loop"
grand_parent: "Building your own harness"
nav_order: 1
---

{% include page_nav.html %}

# Self-improving harness research

As of 2026-11-09. Sub-topic of [The improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}), in [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}).

The [improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}) is a manual practice that agents gradually take over. Through 2026, researchers and vendors tried to automate it end to end: a system that reads its own traces and rewrites its own prompts, skills or tools. This page summarises what that work found, the evidence it adds for enforcing fixes rather than writing more instructions, and what it means for a personal harness.

## The common shape

Across the papers and production reports, the recurring architecture is an outer loop that keeps evidence at every step.

![Sequence diagram of one round of a self-improving harness loop. A proposer agent mines one weakness from traces and makes one small edit to a candidate harness; a protected evaluator runs the tasks and regressions, discards a candidate that regresses, and passes a clean one with its score and diff to a person, who approves it and records the lineage, which becomes the new baseline.]({{ '/assets/images/17-building-a-harness/optimizer-round-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. One round of the common outer loop, with the evaluator and the approval outside what the proposer may edit. Diagram made with archify.*

| Step | What it protects against |
|---|---|
| Capture traces | Fixing from memory instead of from what happened |
| Mine one bounded weakness | Large rewrites that change many things at once |
| Small candidate edit | Changes too big to review or attribute |
| Protected task set | The loop "improving" by changing the test |
| Independent authority | The system approving its own change |
| Lineage and rollback | Being unable to undo a change that looked good |

## Representative work

### Karpathy's autoresearch

A small design pattern for one optimisation task.

| Part | Rule |
|---|---|
| What the agent may edit | One file |
| The evaluator | Protected; the agent cannot change it |
| Budget and metric | Fixed before the run |
| Keep or discard | Decided from repeated trials |
| Setup | Confirmed by a person before the loop starts |

The separation between what the agent may edit and what judges it carries over directly to a personal harness: the agent edits the hook or the skill, and the task set and its checks stay out of its reach.

### Self-Harness and HarnessOpt-Bench

- Self-Harness
  - Mines failures from traces.
  - Proposes edits to the harness.
  - Validates each edit against regressions before keeping it.
- HarnessOpt-Bench
  - Measures how well models do at harness optimisation itself.
  - Uses a trusted, held-out evaluation that the optimiser never sees.

Both put the evaluation protocol at the centre of the method.

### Production practice

OpenAI described building self-improving agents with Codex for a production domain. The progression it reports maps onto the [layers of a harness]({{ '/sections/17-building-a-harness/' | relative_url }}#11-the-four-layers):

1. Feedback from reviewers becomes documentation the agent reads (layer 1).
2. Rules that need enforcement become tooling (layers 2 to 4).
3. Agents carry out both steps, with people reviewing the result.

### The counter-result

One study found that automatic harness changes did no better than simply running the model more. Before trusting an automatic loop, compare it against that baseline:

| Arm | Model | Harness | Budget |
|---|---|---|---|
| Automatic loop | Same | Changed by the loop | Fixed |
| Baseline | Same | Unchanged | Same budget, spent on more runs |

If the two arms score the same, keep the simpler baseline.

## Evidence for enforcing over instructing

Several 2026 studies measured what the [four layers]({{ '/sections/17-building-a-harness/' | relative_url }}#11-the-four-layers) assume: that instructions alone hold less well than checks, and hold worse as they multiply and as a session goes on.

| Study | What it measured | Finding |
|---|---|---|
| TRACE (compiling user corrections into runtime enforcement) | Corrections kept in memory, against corrections compiled into runtime checks | Memory alone: "Mem0 memory still leaves 57.5% of applicable preference checks violated." Compiled checks: "TRACE reduces held-out preference violation from 100.0% to 37.6% on in-distribution tasks and from 100.0% to 2.0% on out-of-distribution tasks." |
| Instruction Stacking Collapse | Follow rate as the number of instructions grows | "Follow rate falls from ∼0.96 at one instruction to 0.604 (Sonnet), 0.433 (Gemini), and 0.201 (GPT-5-mini) at twenty." |
| Instruction adherence in coding agent configuration files | 1,650 Claude Code sessions, varying file size, structure and conflicts | No detectable effect of file structure, but compliance decays within a session: "approximately 5.6% lower odds of compliance per step" |
| Evaluating AGENTS.md | Task success with and without repository context files | Context files "do not generally improve task success rates, while increasing inference cost by over 20% on average" |
| Probe-and-refine tuning of repository guidance | How the context file is produced | A file tuned against real probes reached "33.0% mean resolve rate vs. 28.3% for the static knowledge base used to initialize it and 25.5% for an unguided baseline" |

### What the evidence supports

1. Turn corrections into checks where a program can detect the failure.
   - TRACE's out-of-distribution result is the strongest case, with its caveats below.
2. Keep the number of standing instructions small.
   - The follow-rate drop is per instruction, so every added rule lowers the odds for the others.
3. Expect instructions to fade over a long session.
   - A check runs at the same strength at step 200 as at step 1.
4. Write context files from evidence.
   - The probe-and-refine result favours lines tested against real tasks over a static list.

<details markdown="1">
<summary>Caveats</summary>

- TRACE used a simulated user and 19 hand-curated held-out tasks.
- The configuration-file study found no effect of file size itself; its decay finding is about steps within a session.
- The follow-rate figures come from a benchmark of stacked instructions and vary widely by model.
- I took these figures from my own research survey (sources accessed 2026-09-08) and have not re-read the papers for this page.

</details>

## What this means for a personal harness

### Protect the evaluator

If the agent can change the evaluator, it can make any change look good.

- Keep the task set and the scoring script outside what the loop may edit.
  - In a repository, a separate directory with its own review rule works.
- Review changes to the evaluator separately from changes to the harness.
- Watch for a loop that improves the score by narrowing what the checks look at.

### Your sample is small

With one person's work, the task set is small, and scores are noisy. At 100 tasks with a pass rate near 80 percent, the 95 percent confidence interval is roughly plus or minus 8 percentage points. A change that moves the score by 3 points is inside the noise.

| Tasks in the set | Approximate 95% half-width at an 80% pass rate |
|---|---|
| 20 | about 17 points |
| 50 | about 11 points |
| 100 | about 8 points |

So promote changes on deterministic checks and budgets (did the hook block the bad action, did the run stay under its time limit), and keep a person for questions of taste.

<details markdown="1">
<summary>How the table was computed</summary>

The half-width of a normal-approximation interval for a pass rate p over n tasks is 1.96 times the square root of p(1 - p)/n. For p = 0.8: n = 20 gives 0.175, n = 50 gives 0.111, n = 100 gives 0.078. A Wilson interval gives similar values at these sizes.

</details>

### Keep the person on promotion

The reports agree that the system should not approve its own change. In a personal harness:

1. An agent drafts the fix.
2. The same or another agent writes the test for it.
3. An agent opens the pull request with the test output.
4. You review and merge it.

## Sources

These were gathered by my own research survey in 2026-09 and are not yet verified against the primary sources for this page, except where marked.

- Karpathy, autoresearch: <https://github.com/karpathy/autoresearch>, not yet verified against the primary source.
- Self-Harness: Harnesses That Improve Themselves: <https://arxiv.org/abs/2606.09498>, not yet verified against the primary source.
- HarnessOpt-Bench: Evaluating LLMs at Harness Optimization: <https://arxiv.org/abs/2608.06301>, not yet verified against the primary source.
- OpenAI, Building self-improving tax agents with Codex: <https://openai.com/index/building-self-improving-tax-agents-with-codex/>, not yet verified against the primary source.
- Automatic harness changes doing no better than running the model more: <https://arxiv.org/abs/2607.12227>, verified 2026-11-09.
- TRACE, Getting Better at Working With You: Compiling User Corrections into Runtime Enforcement for Coding Agents: <https://arxiv.org/abs/2606.13174>, not yet verified against the primary source.
- Instruction Stacking Collapse: <https://arxiv.org/html/2608.02639>, not yet verified against the primary source.
- Instruction Adherence in Coding Agent Configuration Files: <https://arxiv.org/abs/2605.10039>, not yet verified against the primary source.
- Evaluating AGENTS.md: <https://arxiv.org/abs/2602.11988>, not yet verified against the primary source.
- Probe-and-Refine Tuning of Repository Guidance for Coding Agents: <https://arxiv.org/abs/2606.20512>, not yet verified against the primary source.

{% include page_nav.html %}
