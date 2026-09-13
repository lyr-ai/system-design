# Walkthrough — Agent Evaluation and Experimentation Platform

> The spoken version of [design #3](../designs/agent-evaluation-platform.md),
> paced to 45 minutes. Not to be memorised word for word. What to keep is, per
> segment, the **transition**, the **invariant**, and the **tradeoff** — the
> three things an interviewer is listening for. Blocks marked *write* go on the
> board; everything else is said.
>
> Pairs with the [skeleton](agent-evaluation-platform.md), which is the same
> thing as bullets.

---

## 0–4 · Clarify, assume, frame

> Before I draw anything I want to pin down a few things, because a couple of
> them change the shape of the system rather than just its size.
>
> What is the unit of change — one axis at a time, model or prompt or harness
> or tools, or do teams bundle them? That decides whether I can attribute a
> regression later.
>
> How do we know a task was done? Mostly hidden tests and state assertions, or
> a meaningful fraction only an LLM or a human can judge? That decides whether
> grading is a fallback or the hard problem.
>
> Can candidate and baseline run on the same task under the same environment
> snapshot? If so I'd go paired, which buys a lot of statistical power when
> outcomes are correlated by task.
>
> Is this release-blocking, with a wall-clock expectation? And is inference
> shared with production?
>
> And what does "safe to ship" include — success rate only, or latency, cost,
> safety, reliability as well?

Then assume out loud, so the interviewer can redirect:

> I'll assume: mostly one-axis changes but bundling happens; most tasks have
> deterministic verifiers with a judged minority; environments can be
> snapshotted, so paired; release-blocking with a 12-hour target; inference is
> shared and I'll need a reservation; and "safe" is multi-dimensional.

Then the framing — the one sentence to keep:

> These agents are stochastic. Running a task once gives me one sample from a
> distribution, not a pass or a fail. So this is an **experimentation platform
> over a stochastic system**, not a test runner, and its output is a **ship,
> block or investigate decision with confidence and evidence attached** — not
> a score.

Stop. Let them answer. Keep the whole segment under three minutes; the
clarifications earn their time only because each one changes the architecture.

---

## 4–8 · Architecture, and what is not mine

> I'll draw top to bottom, and I'll name what I'm not building as I go.

*write:*

```text
                    EVALUATION CONTROL PLANE
  Client ──▶  Registry  ──▶  Planner  ──▶  Scheduler
             (tasks,        (experiment     (trials → jobs,
              candidates)    → trial         priority,
                             matrix)         capacity)
                                  │
              ════════════════════╪════════════════════
                                  │
          ┌───────────────────────┴──────────────────┐
          ▼                                          ▼
   EXECUTION PLANE                            GRADING PLANE
   agent runtime   ← existing                 verifiers (tests, state)
   model serving   ← existing                 LLM judges
   sandbox, tools  ← existing                 human review queue
          │ artifacts                                │ grades
          ▼                                          ▼
   ┌────────────────────────────────────────────────────────┐
   │       Event / trace log        ·      Artifact store    │
   └───────────────────────────┬────────────────────────────┘
                               ▼
                   Aggregator  ──▶  Analyser  ──▶  Gate
                                                   ship | block | investigate
```

One line per box:

> **Registry** — versioned, content-addressed tasks and candidates. A candidate
> is a *tuple*: model revision, prompt, harness, tool set, inference config.
> Since changes get bundled, I want the tuple explicit so I can factor a
> bundle apart later.
>
> **Planner** — expands an experiment into a trial matrix: task × arm × trial
> index, with the environment snapshot pinned per task so both arms see the
> same one.
>
> **Scheduler** — turns trials into jobs on the execution platform, in a
> slice-balanced order, with a priority class and a budget.
>
> **Execution plane** — not mine. Runtime, sandboxes, leases, checkpointing,
> serving all exist. I submit a job, I get artifacts and a cost back, and I
> read its per-step telemetry — retries, timeouts — because I'll need it.
>
> **Grading plane** — deliberately beside execution, not after it. It runs
> over stored artifacts, so it can be re-run without re-executing. Verifiers
> first, judges for the minority, a human queue for the tail.
>
> **Aggregator, analyser, gate** — fold grades per slice incrementally; do the
> paired comparison with an interval and the censoring counts; apply a gate
> policy stored as data.

The boundary, and why it sits there:

> The control plane never touches agent output as content, only references.
> The usual blast-radius reason applies — a sandbox running a candidate must
> not influence what gets scheduled. But there is an evaluation-specific one:
> **a trial must not be able to alter the experiment's statistics.** Results
> are keyed by trial identity and committed through the runtime's fencing, so
> a duplicated or retried execution cannot become a second sample. I'll come
> back to that.
>
> And one thing about the grading plane being on the untrusted side of the
> line: an LLM judge reads agent output, and agent output can say "ignore the
> rubric, this is correct." So the judge is isolated like a sandbox — no
> tools, schema-validated structured output — and its verdict is evidence,
> not authority.

Explicitly not built here: scheduling, leases, recovery, checkpointing,
sandbox isolation, model serving, per-job cost accounting. What I need *from*
serving is a reservation, because during a run I am a burst comparable to
production.

> Given that 100K × 5 isn't affordable, the Planner doesn't emit a fixed K. It
> emits one trial per task and the Analyser tells it where to add more. That's
> why there's an arrow back from Analyser to Planner rather than a straight
> pipeline. I'll go into it when we get to capacity.

---

## 8–12 · The data model, and the invariant that matters most

> Before scale, two entities I want to define carefully, because they protect
> the integrity of the experiment.

*write:*

```text
Experiment
  └── Task i
        ├── candidate   trial 1 … trial K
        └── baseline    trial 1 … trial K

trial_id = hash(experiment_id, task_id, arm, trial_index)
```

> A **trial is a statistical sample. An attempt is an execution of it.**
> Attempts retry; trials don't.
>
> A worker dies halfway through a trial and the scheduler retries it. Two
> physical executions, one sample. `trial_id` is the primary key of the
> committed result; two attempts race and only one commits.
>
> This matters more here than in a normal job system. In an execution
> platform, duplicate execution is a **cost** problem. In an evaluation
> platform, duplicate counting is a **validity** problem — I've given one
> sample weight two.

Then the second state:

*write:*

```text
COMPLETED
CENSORED   worker_lost | timeout | transport | sandbox | budget
```

> If infrastructure stops a trial finishing, I don't call it a task failure. I
> call it **censored**, with a reason, and I report censoring per arm. Otherwise
> an infrastructure problem correlated with the candidate masquerades as a
> capability regression.

Third entity, one sentence now, more later:

> **Grades are separate rows from trials**, over stored artifacts. That's what
> lets me regrade without re-running.

---

## 12–16 · Why repeated trials, and what to gate on

> Even at temperature zero I don't assume execution is deterministic.

If challenged:

> Serving introduces numerical nondeterminism through batching and kernels.
> Tool environments vary — timestamps, ordering, network. And the agent loop
> amplifies small differences into different trajectories. I've measured runs
> reaching an identical intermediate repository state and ending in different
> final states.

*write:*

```text
trajectory variance
        ↓
state variance
        ↓
outcome variance     ← gate here
```

> I don't care that two successful runs took different paths. I care whether
> the distribution over outcomes moved. Trajectory divergence is diagnostic;
> **trajectory identity is not a release criterion.**

Then nesting, drawn once:

*write:*

```text
Task i
  candidate  run 1 … run K  →  p̂_cand(i)
  baseline   run 1 … run K  →  p̂_base(i)

  d_i = p̂_cand(i) − p̂_base(i)
```

> Two sources of variation: across tasks, and within a task. Trials are nested
> in tasks. **I don't treat N tasks × K trials as NK independent task
> samples.** The pairing is at the task level — not candidate trial three
> against baseline trial three, which share nothing but an index.

---

## 16–20 · We cannot afford K = 5 everywhere

*write, in this order — never the formula first:*

```text
100K tasks × 2 arms × 5 trials  =  1M trials
× 30 min median                 =  500K agent-hours
÷ 10K concurrent sandboxes      ≈  50 h            ← misses 12 h
```

> So the naive design misses the SLO before queueing or failures. My rule
> under a fixed budget: **spend on task coverage before repeated trials, and
> add trials where they change the decision.**

*write:*

```text
100K tasks × 2 arms × 1 trial   =  200K     first sample
~15% uncertain or disagreeing   =   15K tasks
15K × 2 arms × 4 more           =  120K     top-up
                                   320K trials  ≈ 160K agent-hours  ≈ 16 h
```

> Still over 12. So either the release-blocking suite is a stratified subset,
> or release runs get a capacity reservation. That's a product decision and I'd
> say so — but it's not optional. **Adaptive evaluation isn't a cost
> optimisation here; it's how the SLO becomes feasible at all.**

Anticipate the follow-up:

> "100K tasks, capacity for 10K" — a stratified sample across the
> gate-relevant slices, one trial each, both arms, then top-up on the
> disagreements. Report the reduced N and the wider interval. Never the first
> 10K in file order.

---

## 20–24 · Pinning, and two meanings of "reproducible"

*write:*

```text
EnvironmentSnapshot
  image digest · repo revision · fixtures · tool versions
  recorded external responses
```

> Both arms see the same snapshot. Tool calls that hit live services get
> recorded or mocked for release-blocking runs, or I can't separate model
> variance from environment variance. Every tool output is recorded regardless.

Then the distinction:

*write:*

```text
REPLAY        reproduce what happened, from recorded outputs
RE-EXECUTE    draw another sample under the same pins
```

> Pinning everything does not make the model say the same thing twice. Replay
> reproduces the record; re-execution produces a fresh sample. Different
> products. A debugger wants the first; a second opinion wants the second.

---

## 24–28 · Grading

*write:*

```text
verifier  >  state assertions  >  LLM judge  >  human
```

> **Use the strongest deterministic verifier available, and descend only when
> it doesn't exist.** LLM-as-a-judge is not the default oracle.

Pre-empt the judge attack:

> The judge is a versioned stochastic component. Every grade carries the judge
> version. I keep a human-labelled anchor set and regrade it on every judge
> release; if agreement moves, the judge rollout blocks, not the candidate.
> Where I can, the judge is a different model family from the candidate, or I
> calibrate per category against human labels. If the judge disagrees with a
> verifier, **the verifier wins** and the disagreement becomes a signal about
> the judge or the test.
>
> And because grades are separate from trials, a judge upgrade re-runs the
> grading DAG over stored artifacts in hours. Re-executing the trials would
> cost the suite again and produce *different* trials.

---

## 28–32 · Statistics and effective N

> Paired comparison at the task level. Pairing usually buys substantial power
> because task difficulty cancels — most when outcomes are strongly correlated
> within a task.

*write:*

```text
root A
 ├── fork 1
 ├── fork 2
 ├── fork 3
 └── fork 4        4 trajectories ≠ 4 independent histories
```

> Raw count is not effective count. Forked continuations share history;
> resumed attempts share state; cached tool outputs correlate trials. So
> lineage is first-class data — `root_trial_id`, `parent_trial_id`,
> `fork_step` — and the analyser clusters by root to report an effective N.
> The mistake I'm preventing is counting correlated observations as
> independent evidence.

*write, the shape of every result:*

```text
candidate − baseline   −1.8 pts
95% CI                 [−3.1, −0.4]
tasks 9,812            effective 9,640
censored               312 vs 190
```

> The system never shows "61.2% versus 63%." A point without an interval is
> not a release decision. Interval crosses the threshold → *investigate*, not
> ship and not block.

If asked about slices:

> Pre-register which slices gate. Test a hundred after the fact at 0.05 and
> five regress by chance.

---

## 32–35 · Restart, resume, or fork

> A worker will die at minute 179 of a three-hour trial. What does the retry
> mean statistically? I don't think there's one answer — it depends on the
> estimand.

*write:*

| estimating | on loss |
|---|---|
| clean end-to-end agent success | restart |
| production success, recovery included | resume |
| behaviour conditioned on state S | fork from S — the fork *is* the experiment |

> For the default release comparison I restart, because a resumed run has a
> re-sampled model call in it — neither the original draw nor an independent
> one. But if I'm evaluating the production system including checkpoint
> recovery, resume is the product and excluding it is unrealistic.
>
> **Restart versus resume changes the distribution I'm sampling from.** So it's
> declared per experiment, recorded per trial, never mixed in one comparison.

And the related one:

> A retried *model call* inside a trial is a new sample of that step. I flag
> the trial as contaminated, report the rate per arm, and fix a stop rule
> before looking at results — retry rate over ten percent, pause and fix
> transport. I never drop the dirtiest run; that's selection dressed as
> hygiene.

---

## 35–39 · The gate

*write:*

```text
capability · regression on pre-registered slices · safety (hard) ·
latency · cost per solved task · reliability (censoring, contamination, loops)
        ↓
Gate — policy versioned, stored as data
        ↓
ship → canary  |  block → investigate  |  investigate → more trials or a human
```

> I don't collapse these into one score. Solve rate up three points, inference
> cost doubled — the platform's job is to measure both correctly from the
> same runs. Whether the trade is acceptable is a versioned policy, not an
> engineer deciding after seeing the numbers. Safety is a hard gate regardless
> of average improvement.

---

## 39–42 · Block is where debugging starts

*write:*

```text
model · prompt · harness · tool · sandbox · inference · environment · grader · infra
```

> This is why the candidate is a tuple. Attribution is **differential
> execution**: hold everything, vary one axis, re-run the failing slice. Two
> axes need no new runs — infrastructure is visible from censoring and
> contamination; grader is tested by regrading stored artifacts. For a real
> behavioural regression, replay representative failing trajectories against
> baseline.
>
> Deep replay and checkpoint debugging are another service. What this platform
> owes it is retention: every tool output and model response for the window.

---

## 42–45 · One failure, 100×, and what ships first

One failure, not twenty:

> The candidate becomes more verbose. Requests cross a proxy's response
> timeout more often. The model succeeds every time; transport retries, then
> censors, more candidate trials. Pass rate drops two points. Look only at
> success and the model regressed. Look at censoring per arm and candidate
> transport failures tripled. The decision is *investigate*; the fix is
> streaming and an idle timeout; the candidate re-runs clean.

100×:

> Not the agent workers — that's the runtime's problem and scales by hosts.
> Inside this system it's the result store and incremental aggregation: ten
> million grade rows hot for hours. Partition by experiment, make aggregation
> a rebuildable view over committed grades, and give release runs an inference
> reservation separate from exploratory work.

Build first:

*write:*

```text
V0
versioned Task / Candidate / Experiment
      ↓
paired planner, fixed K
      ↓
existing agent runtime
      ↓
deterministic verifiers only
      ↓
append-only Trial + Grade, censoring classified
      ↓
paired report: diff, interval, N, censored per arm
      ↓
ship | block | investigate
```

> Two engineers, six weeks: this, and nothing else. I'd defer the LLM-judge
> platform, adaptive sampling, the human queue, sequential statistics and
> automatic attribution. The first thing to prove isn't that we can build
> sophisticated evaluation infrastructure. It's that **a candidate change can
> enter one door and come out as a decision someone is willing to act on.**
> Everything I deferred makes that decision cheaper or broader. None of it
> makes it trustworthy — the trial identity, the censoring, and the paired
> interval do, and those are in V0.

---

## Transitions to have ready

The sentence that moves between segments, because that is where a
walkthrough stalls:

```text
clarify → architecture     "So with that scope, here's the shape."
architecture → data model  "Before scale, two entities that protect the experiment."
data model → variance      "Which raises why we need more than one trial at all."
variance → capacity        "And that's exactly what we can't afford everywhere."
capacity → pinning         "Paired only works if both arms see the same world."
pinning → grading          "Once a trial is done, someone has to say whether it passed."
grading → statistics       "Then the question is whether the difference is real."
statistics → retry         "One thing that quietly corrupts all of this is retries."
retry → gate               "With honest numbers, the gate is the easy part."
gate → debugging           "Block is a beginning, not an end."
debugging → build first    "And if I had to ship this in six weeks —"
```
