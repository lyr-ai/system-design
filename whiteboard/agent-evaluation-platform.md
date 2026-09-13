# Whiteboard — Agent Evaluation and Experimentation Platform

> The 45-minute version of [design #3](../designs/agent-evaluation-platform.md).
> Only what to say unprompted. Section numbers point back at the full text.

---

## 0–5 · Frame it (say these three things, then stop)

1. **Stochastic subject.** A run is one draw. The unit of evaluation is a
   distribution; the output is a *decision with a confidence*, not a score.
2. **Composes.** #1 runs a trial, #2 serves the model. This design is
   orchestration, grading, statistics, the gate — and what happens after *block*.
3. **Scope.** Coding/tool agents · verifier first, judge as fallback · paired
   design on a pinned snapshot · release-blocking, 12 h · reproduce for 30 days.

Clarify only: what varies (model / harness / prompt / tools)? verifiable or
judged? paired possible? does it block a release? shared inference?

---

## 5–10 · Draw (in this order)

```text
 Control plane:  Registry → Planner → Scheduler          (never touches content)
                        │ trial matrix
        ┌───────────────┴───────────────┐
   Execution plane                 Grading plane          (side by side — grading
   Agent runtime #1 ── artifacts ─▶ verifier / judge / human   is a re-runnable stage
   Model #2 · Tools · Sandbox                                  over stored artifacts)
        └──────── Event log · Artifact store ────────┘
                              ↓
                Aggregator → Analyser → Gate → ship | block | investigate
```

Entities, one line each: **Task** (versioned, content-addressed) · **Candidate**
(a *tuple*: model, serving, harness, prompt, tools, inference config) ·
**Experiment** · **Trial** (one sample) · **Grade** (separate row, regradable) ·
**Decision**.

---

## 10–17 · The trial matrix   [§8]

```text
Task × Candidate × EnvironmentSnapshot × K trials
```

- Temperature 0 still needs repeats: serving, environment, and the agent
  amplifying both. Measured: same state → different final states.
- Gate on the **outcome distribution**; trajectory variance is diagnostic only.
- Interval half-width ≈ 1.96·√(p(1−p)/N). N that generalises is **tasks**.
  → *Spend on tasks before trials; add trials where the arms disagree.*
- Adaptive: 1 trial everywhere → top up disagreements → stop when the interval
  clears the threshold (with a sequential correction).
- Pin the environment; record every tool output.

---

## 17–25 · Integrity   [§9]  — the section that is not #1

**Trial ≠ attempt.** Trial = sample, attempt = execution.
`trial_id = hash(experiment, task, arm, index)` is the result's primary key;
the attempt fence discards the loser. *Duplicate execution is a cost problem
in #1 and a validity problem here.*

**Censored ≠ failed.** Infra failure → `CENSORED` with a reason, out of the
denominator, **reported per arm**. A candidate that times out more is a finding,
not a fail.

**Retried model call = contaminated.** Flag, report the rate per arm, stop rule
fixed in advance (retry rate > 10%, a step resampled > 3×). Never drop the
dirtiest run.

**Restart or resume — depends on the estimand.**

| estimating | on loss |
|---|---|
| clean end-to-end success | restart |
| production success incl. recovery | resume |
| behaviour from state S | fork *is* the experiment |

"It changes which distribution I'm sampling from; declared per experiment,
recorded per trial."

---

## 25–31 · Grading   [§10]

```text
verifier  >  state assertions  >  LLM judge  >  human
```

Descend only when the stronger one does not exist. The judge is a **versioned
stochastic component**: sample it K_j times · `grader_version` on every grade ·
anchor set with human labels re-graded on every judge release · different
family from the candidate or calibrate · **verifier wins** disagreements (the
disagreement is a signal about the judge or the test) · isolate it like a
sandbox — agent output is untrusted text.

Grades separate from trials → **regrade without re-execute**.

---

## 31–37 · Statistics   [§11]

- **Paired**: per-task difference; discordant pairs carry the information.
  Gain is large when outcomes correlate within task — usually, not always.
- **Nested**: trials within tasks. `d_i = p̂_cand(i) − p̂_base(i)`; the task
  pairs, not trial 3 with trial 3. **NK trials are not NK independent samples.**
- **Effective N from lineage**: `root / parent / fork_step` → cluster by root.
  20 trajectories from 4 roots ≠ 20.
- **Report the interval, never the point**: diff, CI, N, effective N, censored
  per arm, contaminated per arm. Interval crosses the threshold → *investigate*.
- Pre-register gate slices; the rest is exploratory. Peeking needs a spending
  function.

---

## 37–42 · Gate and after *block*   [§12]

Six dimensions, policy is data: capability · regression · safety · latency ·
cost · reliability (censoring, contamination, loops). *3 points up, 2× cost?*
→ the gate has a cost dimension; the platform shows both from the same runs;
the policy weighs them.

**Attribution = differential execution.** Candidate is a tuple → hold baseline,
vary one axis, re-run the failing slice. Infra and grader axes need no new
runs (classification; regrade). Replay from recorded outputs for the debugger;
re-execution is a new sample.

---

## 42–45 · Numbers and the trade

```text
100K × 2 arms × 1        = 200K      first sample
15K uncertain × 2 × 4    = 120K      top-up
                           320K trials × 30 min / 10K sandboxes ≈ 16 h  > 12 h
```

→ stratified 50K for release-blocking, or a capacity reservation. Product
decision; say which. Inference during a run ≈ #1's whole production load, so
evaluation gets its own reservation.

**Build first:** trial identity + lineage schema; grades separate from trials.
V0 = versioned Task/Candidate/Experiment → paired planner, fixed K → #1 →
verifiers only → append-only Trial + Grade → paired report. Defer judge
platform, adaptive K, human queue, attribution. *Prove one door in, one
trustworthy decision out.*

---

## Do not

- Open with LLM-as-a-judge.
- Say "halves the sample size" without the correlation condition.
- Say "restart, always" — say what is being estimated.
- Put a formula on the board before the three-line derivation.
- Report a percentage without N, interval and censored count.
