# System Design #6 — Post-Training and Fine-Tuning Platform

> **Design a multi-tenant platform for fine-tuning and post-training large
> language models: from dataset and experiment definition through distributed
> training, evaluation, promotion, and hand-off to inference.**

The concrete case: a team has a 27B open-weight base model, a few hundred
thousand curated coding-agent trajectories, and a preference dataset. They
want to run twenty experiments this week — supervised fine-tunes, LoRA
variants, a preference-optimisation pass — evaluate every checkpoint against
production, and promote one. Next month they want to try reinforcement
learning with the agent itself generating the rollouts.

This is deliberately not frontier pre-training. The scale is:

```text
typical job         8–64 GPUs
large job           128–256 GPUs
duration            hours to days
methods             SFT, LoRA / PEFT, preference optimisation (DPO-style)
occasionally        RL: rollout → reward / verifier → update
```

**The framing that decides everything else:** this is a **lineage and
scheduling** problem whose workload happens to be training. The hard parts are
not the matrix multiplies — a framework does those — but knowing exactly which
data, code, base model and seed produced a checkpoint; getting 64 GPUs at
once on a pool shared with inference; surviving one of them dying at hour
nine; and turning a directory of checkpoints into a decision.

It composes with the rest of the series, and the composition is the design:

```text
Dataset ──▶ #6 post-training ──▶ candidate checkpoint ──▶ #3 evaluation ──▶ promote
                 │                                             ▲
                 │  RL rollouts run on                          │ verifiers, sandboxes
                 └────▶ #1 agent runtime · #4 sandboxes · #2 inference
                                                                │
                                              promoted model ──▶ #2 inference
```

> I won't design the inference platform, the evaluation platform, or the
> sandbox here — those are #2, #3, #4. This design is what happens *before*
> a model reaches inference, and it treats those three as services it calls.

---

## 1. Problem statement

A client submits:

```text
experiment
  base_model          by digest
  dataset             by version
  recipe              method (SFT | LoRA | DPO | RL), code revision, hyperparameters, seed
  resources           GPUs, topology preference, priority, deadline
  eval policy         which suite, at which steps, what promotes
```

The platform:

```text
admits and schedules a gang of GPUs
materialises data and base weights on the workers
trains, checkpointing durably
evaluates checkpoints as they land, via #3
records lineage from dataset to candidate
promotes a candidate to a registry #2 can load
```

Must support:

```text
tens of concurrent experiments, hundreds of GPUs, shared with inference
full fine-tune and parameter-efficient methods on the same substrate
restart from a checkpoint after any single failure, losing bounded work
every checkpoint traceable to data version, code, base, seed, step
eval-in-the-loop: continue, stop early, or promote without a human polling
the RL loop, where the rollout workers are agents
```

---

## 2. Clarify before designing

| question | what it changes |
|---|---|
| Full fine-tune, PEFT, or both? | memory per GPU by an order of magnitude; whether sharding is mandatory |
| Largest model, and largest job? | whether tensor/pipeline parallelism is in scope at all — here, mostly not |
| GPU pool shared with inference? | preemption, reservations, and the diurnal trade |
| Is RL in scope? | a second fleet (rollouts), weight synchronisation, and a staleness policy |
| Who owns evaluation? | if #3 exists, this platform emits checkpoints and consumes decisions |
| Reproducibility target? | bitwise is unrealistic across hardware; same-data-same-order is achievable and is the bar |
| Data governance? | dataset versioning, access control, and the lineage that proves what a model saw |
| Multi-tenant across teams? | quotas in GPU-hours, priority classes, fair queueing |

**State the scope out loud:**

> I'll assume models up to ~30B, jobs up to 256 GPUs, full fine-tune and LoRA
> both supported, a pool shared with inference, RL in scope but not the
> common case, #3 owning evaluation, and reproducibility meaning same data in
> the same order under the same code — not bitwise.

---

## 3. Non-functional requirements

### Lineage — the requirement that distinguishes this from a job runner

```text
any checkpoint → dataset version, code revision, base digest, recipe, seed, step
any promoted model → the checkpoint, and the #3 decision that promoted it
```

### Reliability

```text
single GPU / host failure       restart from checkpoint, ≤ one interval lost
checkpoint publish              atomic; a partial checkpoint is never visible
```

### Throughput

Measured in **tokens per second per GPU** and **MFU**, not GPU utilisation —
utilisation reads 100% while a job waits on all-reduce.

### Scheduling

```text
gang: all GPUs or none        queue wait reported honestly
preemption only to a checkpoint
```

### Reproducibility

Same run spec → same data order, same initialisation, same hyperparameters.
Loss curves that agree closely; not identical bits.

---

## 4. Data model

Six entities.

### Dataset — versioned and content-addressed

```text
Dataset {
  dataset_id, version            hash of the manifest
  shards[]                       content-addressed, tokenised or raw
  schema, tokenizer_digest
  provenance                     where the rows came from; filters applied
  access                         who may train on it
}
```

Editing a filter produces a new version. A model trained on version 7 says
so forever.

### BaseModel

```text
BaseModel { model_id, weights_digest, architecture, tokenizer_digest, license }
```

### Recipe — everything that is not data or base

```text
Recipe {
  recipe_id                      hash of the below
  method                         SFT | LoRA | DPO | RL
  code_revision
  hyperparameters                lr schedule, batch, seq len, accumulation, precision
  parallelism                    dp, fsdp shard degree, tp, pp
  seed
  environment_digest             container, CUDA, framework versions
}
```

### Run — one attempt at an experiment

```text
Run {
  run_id, experiment_id, attempt
  dataset_version, base_digest, recipe_id
  gang                           GPUs, hosts, topology
  status
  resumed_from_checkpoint
}
```

### Checkpoint — a training checkpoint is not an agent checkpoint

```text
TrainingCheckpoint {
  checkpoint_id, run_id, step
  weights_ref                    sharded, content-addressed
  optimizer_ref                  sharded — often larger than the weights
  data_cursor                    shard, offset, epoch — where the loader was
  rng_state                      per rank
  lr_schedule_state
  metrics_at_step
  manifest                       published last, atomically
}
```

Two fields that are easy to leave out and expensive to have left out:
**`data_cursor`**, without which a restart re-reads data it already trained on
(§18), and **`optimizer_ref`**, without which a restart is a different
optimisation trajectory that happens to share weights.

### CandidateModel — a tuple, not a file

```text
CandidateModel {
  candidate_id                   hash of (base, dataset_version, recipe, step)
  checkpoint_id
  export_ref                     merged / converted weights #2 can load
  eval_decisions[]               from #3
  promotion_status
}
```

The registry is a lineage graph with files hanging off it, not a file store
with tags.

---

## 5. High-level architecture

```text
              ┌────────────────────────────────────────────┐
              │           TRAINING CONTROL PLANE           │
  Client ────▶│  Experiment API · Registry (datasets, bases,│
              │  recipes, candidates) · Gang Scheduler ·    │
              │  Lineage · Eval Orchestrator · Promotion    │
              └───────────────────┬────────────────────────┘
                                  │ gang of N GPUs, run spec
                                  ▼
    ┌───────────────────────────────────────────────────────┐
    │                    TRAINING WORKERS                    │
    │   rank 0 … rank N-1   ·  data loader  ·  checkpointer  │
    │   collectives over NVLink (intra-node) / IB (inter)    │
    └───────────┬───────────────────────┬───────────────────┘
                │ checkpoints           │ metrics, health
                ▼                       ▼
     Checkpoint / Artifact Store     Telemetry
                │
                ▼
     Eval Orchestrator ──▶ #3 Evaluation ──▶ decision ──▶ Promotion ──▶ #2

    RL only:
    Trainer ──weights──▶ Rollout Fleet (#2-style serving) ──▶ #1 agents in #4 sandboxes
       ▲                                                          │
       └──────────── trajectories + rewards (verifiers, #3) ◀─────┘
```

Draw the training path first; add the RL loop as a second diagram only if
asked. The boxes that are *not* here: the model-serving engine, the grading
hierarchy, the sandbox. They are called.

---

## 6. Control plane vs data plane

| | control plane | data plane |
|---|---|---|
| owns | experiments, registry, lineage, gang admission, eval orchestration, promotion | training workers, data loading, collectives, checkpoint writing, rollout workers |
| runs user code | never — recipes are executed on workers | always: the training script is user code |
| failure | experiments stall | one run loses ≤ one checkpoint interval |
| trust | authenticated callers | the run's identity, fenced by attempt |

Training code is user code. It can be wrong, slow, or malicious, and it runs
with GPUs and a dataset it was granted. So checkpoints are published under
the run's attempt (fencing, exactly as in #1), datasets are mounted read-only
by version, and the registry accepts a candidate only from the run that
lineage says produced it.

---

## 7. Lifecycle — one SFT job, step by step

Before parallelism, what one job actually does:

```text
SUBMITTED
  ↓ resolve dataset version, base digest, recipe; compute lineage id
ADMITTED
  ↓ quota, priority, gang request queued
SCHEDULED
  ↓ N GPUs allocated together, topology-aware (§9)
MATERIALISING
  ↓ workers pull base weights (once per host, cached); loader opens shards at cursor 0
TRAINING
  │  loop:  load batch → forward → backward → all-reduce grads → optimizer step
  │         every K steps: checkpoint (async, §10)
  │         every E steps: export candidate → eval orchestrator → #3
  ↓
COMPLETED | STOPPED_EARLY | PROMOTED
```

Crash path:

```text
rank 17's GPU faults
  ↓ collective times out on every rank; the gang is dead, not one worker
  ↓ run attempt++ ; scheduler reallocates the gang (possibly different hosts)
  ↓ all ranks restore weights, optimizer, data cursor, RNG from the last committed checkpoint
  ↓ resume at step S+1
```

The line that separates this from #1: **a gang fails as a unit.** One GPU
lost means N GPUs idle until the run is restarted, which is why checkpoint
frequency and gang restart latency are the reliability story.

---

## 8. Deep dive — why a job cannot fit on one GPU, and what to do about it

Start from the memory arithmetic; everything else follows.

### Full fine-tune, mixed precision, Adam

Per parameter, roughly:

```text
weights (bf16)             2 B
gradients (bf16)           2 B
master weights (fp32)      4 B
Adam m, v (fp32)           8 B
                          ────
                         ~16 B / parameter, before activations
```

For the 27B model:

```text
27B × 16 B  ≈  430 GB           of *state*, before a single activation
```

That does not fit on an 80 GB GPU. It does not fit on four. So the question
is never "how many GPUs for speed" first; it is "how many GPUs for the state
to exist at all".

### Data parallelism alone is not enough

DDP replicates the full state on every GPU and all-reduces gradients. It
scales *throughput*; it does nothing for *memory*. At 430 GB of state per
replica it cannot start.

### Shard the state: FSDP / ZeRO

```text
ZeRO-1   shard optimizer state         16 B → ~8 B  per param per GPU
ZeRO-2   + shard gradients             → ~6 B
ZeRO-3 / FSDP   + shard weights        → ~16 B / N   per GPU, gathered per layer on demand
```

With FSDP across 16 GPUs:

```text
430 GB / 16  ≈  27 GB per GPU of state
+ activations (sequence length × batch × layers; checkpointed to recompute)
```

fits on 80 GB with room. The cost is communication: each layer's weights are
all-gathered before use and freed after, every step. That traffic is what
NVLink inside a node and InfiniBand across nodes are for, and it is why
placement cares about topology (§9).

### Parameter-efficient methods change the arithmetic entirely

LoRA freezes the base weights and trains small adapters:

```text
base weights (bf16, frozen)    54 GB      no gradients, no optimizer state
adapter parameters             ~50M       ~1 GB with optimizer state
```

The state fits on one or two GPUs; data parallelism is enough; the base can
even stay in FP8 (27 GB). LoRA is not a smaller version of fine-tuning — it is
a different resource profile, and the platform should schedule it as one.

### Where tensor and pipeline parallelism come in — and why not here

TP splits individual matrices across GPUs and adds an all-reduce per layer per
step; PP splits layers into stages and adds bubbles. Both exist because at
70B–400B even a sharded state and per-layer gathers become the bottleneck.
At ≤30B with FSDP, they are not needed, and adding them is complexity with
no return. Know why they exist; do not reach for them.

### The knobs, and what they trade

```text
gradient accumulation    larger effective batch without memory; slower per step
activation checkpointing ~30% more compute for a large memory saving
mixed precision          bf16 compute, fp32 master — the default, not an option
sequence packing         fills padding with real tokens; the cheapest throughput win
```

---

## 9. Deep dive — gang scheduling on a pool shared with inference

> This is the "capacity with a shared GPU pool" deep dive that #1 §8 and
> §17 pointed at.

### All or none

A 64-GPU FSDP job with 63 GPUs is a job with 0 GPUs: every step is a
collective. So admission is **gang** admission — the scheduler holds a
reservation until all N are free, then starts all N.

The consequence is fragmentation: a pool with 60 free GPUs scattered across
hosts cannot start a 64-GPU job, and cannot start an 8-GPU job either if
those 60 are one-per-host and the job wants NVLink locality. The scheduler's
job is to keep gangs *packable*:

```text
place small jobs to preserve whole nodes
backfill short jobs into gaps while a large gang waits
drain, do not scatter
```

### Topology is a hard constraint, not a preference

```text
intra-node    NVLink / NVSwitch        ~900 GB/s
inter-node    InfiniBand               ~50–400 GB/s per host, shared
```

FSDP gathers per layer every step. An 8-GPU job split across two hosts runs
at a fraction of the speed of one on a single node, and the difference is
not tunable away. Placement therefore treats *node-aligned* as required for
jobs ≤ 8 GPUs, and *rack-aligned* as strongly preferred above that.

### The shared pool with inference

Inference (#2) is diurnal; training is not. The honest policy:

```text
inference    reserved floor for its SLO, autoscaled above it
training     everything above the floor, by priority
night        the floor shrinks; training gangs grow
preemption   low-priority training → to a checkpoint; never inference
```

Preemption to a checkpoint is what makes training the elastic partner. A job
preempted at step 4,000 with a checkpoint at 3,900 loses 100 steps and its
gang; it re-queues. Without checkpointing, preemption is destruction, and
sharing the pool is impossible.

### Queue wait is a first-class metric

Report it. A platform where a 64-GPU job waits nine hours and the team does
not know why is a platform people route around with private clusters.

---

## 10. Deep dive — failure recovery when one of 64 dies

### What a checkpoint must contain

```text
weights          sharded, per rank
optimizer state  sharded — for Adam, twice the weights
data cursor      shard index, offset, epoch, per rank
RNG state        per rank, including dropout and data shuffle
LR schedule      step, warmup state
```

Weights alone restore *a* model, not *the run*. Missing optimizer state
resets Adam's moments and changes the trajectory; missing the data cursor
re-trains on seen data (§18); missing RNG breaks same-data-same-order.

### Frequency is the same trade as in #1

```text
expected loss   ≈  failure rate × interval / 2  (× N GPUs, all idle)
checkpoint cost ≈  size / write bandwidth, per interval
```

For the 27B full fine-tune:

```text
state 430 GB; object store at ~10 GB/s aggregate     → ~45 s per checkpoint
64 GPUs at ~$2/h                                     → 45 s costs ~$1.60
a failure every ~day per 64 GPUs, 30-minute interval → ~15 min × 64 GPUs lost, ~$32
```

So every 15–30 minutes is reasonable, **provided the write is asynchronous**:
copy the shards to host memory, let training continue, upload in the
background, publish the manifest last. A synchronous 45-second stall every 15
minutes is a 5% throughput tax; asynchronous, it is nearly free.

### Publish atomically, fenced by attempt

Same shape as the agent checkpoint: shards are immutable, content-addressed,
written by every rank; the manifest is one small write that appears only
after every shard has landed, and it carries the run's attempt so a
straggling rank from a dead attempt cannot publish over the live one.

### Detecting the failure

A dead GPU does not announce itself; the collective hangs. So collectives run
with a timeout, and a timeout is treated as *gang dead*, not *retry*. Then:

```text
mark attempt failed → release the gang → re-queue at head with reservation
→ new gang (different hosts allowed) → restore → continue
```

Restart latency is dominated by re-materialising base weights on new hosts
(cached per host) and restoring 430 GB of state (~45 s from object store, or
seconds from a host-local copy if the hosts survived).

### Stragglers

One slow GPU — thermal throttling, a bad link — slows the whole gang, since
every step waits for the slowest rank. Detect by per-rank step time; a rank
consistently >20% behind is evicted and the gang restarted on healthy
hardware. Ten minutes of restart beats hours at 60% speed.

---

## 11. Deep dive — lineage and evaluation in the loop

### The candidate is a tuple

```text
candidate_id = hash(base_digest, dataset_version, recipe_id, step)
```

so that *"the 27B SFT from Tuesday"* is never a name. Two runs with the same
tuple are the same candidate re-trained; the platform can say whether their
evals agree, which is the reproducibility check that matters.

### Do not wait for the run to finish

Twenty experiments, each producing a checkpoint every 500 steps, is not a
directory a human inspects on Friday. The eval orchestrator:

```text
checkpoint published at step S
      ↓ export (merge LoRA, convert) → candidate
      ↓ submit to #3: a small fast suite every checkpoint, the full suite on a cadence
      ↓ decision arrives
      │
      ├── continue
      ├── stop early      quality flat or declining for M evals; free the gang
      └── promote         gate passed; candidate → registry → #2 can load it
```

Early stopping is a *scheduling* feature: it returns GPUs. On a shared pool
it is the difference between twenty experiments a week and eight.

### What #3 needs from #6

The candidate tuple, so its attribution works (which axis changed — data,
recipe, base, step); the export in a form #2 serves; and cost per candidate,
which is GPU-hours from the lineage. What #6 needs from #3: a decision with
its interval, per candidate, and the failing slices when it blocks — which
feed the next recipe.

### The registry is a graph

```text
base ──▶ run (dataset v7, recipe R, seed 3) ──▶ ckpt@500 ──▶ ckpt@1000 ──▶ promoted
                                                    │
                                                    └──▶ eval decisions
```

Queries the registry must answer: *what did this model see*, *which
experiments used dataset v7*, *what is in production and what produced it*,
*what changed between the two candidates we are comparing*. A tag on a file
answers none of them.

---

## 12. Deep dive — the RL loop, where rollouts are agents

For preference and RL post-training the data is not a static dataset. It is
generated by the model being trained:

```text
        policy checkpoint θ_k
              │  weights
              ▼
      Rollout fleet (serving, #2-style)
              │  prompts → completions / agent trajectories (#1, in #4 sandboxes)
              ▼
      Reward / verifier (#3's grading hierarchy: tests > assertions > judge)
              │  (trajectory, reward, policy_version = k)
              ▼
      Trainer ──▶ θ_{k+1} ──▶ back to the rollout fleet
```

Everything from #1, #3 and #4 shows up here at once, and four systems
questions decide whether the loop runs at all.

### Rollouts are the bottleneck, by arithmetic

```text
one update:   1,000 prompts × 8 samples × 2,000 tokens  =  16M generated tokens
serving at ~1,000 tokens/s per GPU (batched decode)
  64 GPUs                                                ≈  250 s
trainer step on 16M tokens, 27B, 64 GPUs at 40% MFU      ≈  100 s
```

Generation costs more than training on it. So the rollout fleet is larger
than the training gang, and **the loop's throughput is decode throughput** —
every lesson from #2 applies: batching, prefix caching on shared prompts,
KV-bounded capacity.

### Staleness: which policy generated this trajectory?

The trainer finishes θ_{k+1} while rollouts from θ_k are still arriving. Use
them, and the update is off-policy by one step; discard them, and 250 s of
GPU time is thrown away. The platform does not decide — it **records**
`policy_version` on every trajectory and lets the recipe declare a staleness
bound (`≤ 1 version behind` is the usual answer). The systems requirement is
the field, not the policy.

### Synchronising weights

θ_{k+1} is 54 GB in bf16. Pushing it to a 64-GPU rollout fleet every update
is 3.5 TB of transfer per step — too slow through object storage, fine over
the training fabric with a broadcast, or cheap if the rollout fleet serves
LoRA adapters over a fixed base (megabytes per update). This is where LoRA
buys its largest win: not memory, but **update latency in the loop**.

### Rollouts on #2, or a dedicated fleet?

Sharing #2's production replicas means the RL loop competes with users and
its weight updates churn their prefix caches. A dedicated rollout fleet, using
#2's engine and #2's scheduling logic, is the usual answer — same software,
separate capacity, updated on the loop's cadence.

### And the verifiers are #3's

Reward for a coding rollout is *did the tests pass*, run in a #4 sandbox with
no effects; a judge only where no verifier exists, with the same version
pinning and calibration #3 requires. A reward model is a grader with all of a
grader's failure modes, and reward hacking is the name for a candidate
optimising against the judge's weaknesses — which is why the deterministic
verifier sits at the top of the hierarchy here too.

---

## 13. Data pipeline

The GPU must never wait for data. The pipeline is boring by design:

```text
raw → filtered → tokenised → packed to sequence length → sharded → shuffled per seed
      each stage content-addressed; the dataset version is the hash of the final manifest
```

At train time:

```text
each rank owns a slice of shards; prefetches ahead of the step
shuffle order is a function of (seed, epoch) — reproducible, and restorable from the cursor
```

Tokenise offline, once per dataset version, never in the training loop. Pack
sequences so padding does not consume the batch. Mount shards read-only by
version; a training job cannot mutate its dataset, and the lineage stays true.

---

## 14. Multi-tenancy and quotas

```text
per team      GPU-hours per period; max concurrent GPUs; priority class
classes       production post-training > scheduled experiments > exploratory
preemption    exploratory → to checkpoint; never a promotion-track run mid-eval
fairness      weighted by class over *allocated GPU-time*, not job count
```

A 256-GPU job and thirty-two 8-GPU jobs are the same capacity; fairness that
counts jobs lets one team monopolise the pool with many small ones.

---

## 15. Observability

```text
per run       tokens/s per GPU · MFU · step time and its per-rank spread ·
              data-loader wait · checkpoint stall · loss and eval curves ·
              queue wait before start
per pool      allocation vs reservation · fragmentation (largest schedulable gang) ·
              preemptions · restart latency · straggler evictions
RL            rollout tokens/s · staleness distribution · reward by verifier class ·
              weight-sync latency
```

**GPU utilisation is not a training metric.** It reads high while ranks wait
on a collective or a slow loader. MFU and tokens/s per GPU are what say
whether the hardware is doing work.

---

## 16. Storage

```text
datasets        object store, content-addressed shards; read-only mounts by version
base weights    object store; cached per host; the largest cache win
checkpoints     object store, sharded, content-addressed; retention by lineage —
                promoted and their ancestors kept; exploratory tiered fast
exports         merged / converted weights in the format #2 loads
registry        metadata DB: the lineage graph
```

Retention follows lineage, not age: a checkpoint that a promoted model
descends from is kept as long as the model is in production, because it is
the only way to answer *what produced this*.

---

## 17. Capacity estimation

### One SFT run

```text
27B model, 1B tokens of SFT
FLOPs ≈ 6 × params × tokens  =  6 × 27e9 × 1e9  ≈  1.6e20
H100-class at ~1e15 dense bf16, 40% MFU           ≈  4e14 effective per GPU
64 GPUs                                            ≈  2.6e16 / s
                                                   ≈  6,300 s  ≈  1.7 h
```

LoRA on the same data: roughly two-thirds of the FLOPs (no weight-gradient
matmuls for frozen layers) on far fewer GPUs — the same 1.7 hours on 8 GPUs
is not far off, because the base still has to be forwarded and backwarded.

### The pool

```text
512 GPUs, shared
inference floor by day     ~256      night  ~128
training capacity          ~256 by day, ~384 by night
20 experiments/week at 64 GPU × 2 h  =  2,560 GPU-hours  ≈  13% of a week's training capacity
```

Twenty experiments is not the limit; queue wait and gang fragmentation are.

### Checkpoints

```text
full state 430 GB × every 30 min × 1.7 h   ≈  4 checkpoints per run, ~1.7 TB
20 runs/week                                ≈  35 TB/week, tiered aggressively
LoRA checkpoints                            ≈  1 GB each — negligible
```

### The RL number that changes the shape

```text
rollout   16M tokens / 64 GPUs   ≈  250 s
train     16M tokens / 64 GPUs   ≈  100 s
```

The rollout fleet needs ~2.5× the training gang to keep the trainer busy —
or the loop runs at 40% trainer utilisation. Generation is the cost of RL
post-training, and it is an inference-shaped cost.

---

## 18. Failure modes

| failure | handling |
|---|---|
| GPU fault mid-run | collective timeout → gang dead → restart from checkpoint on a new gang |
| host lost | same; base weights re-pulled on replacement hosts |
| straggler rank | per-rank step time; evict and restart |
| checkpoint partially written | manifest never published; previous checkpoint used |
| stale rank publishes after restart | attempt fencing on the manifest |
| dataset version deleted under a run | forbidden: retention follows lineage |
| eval service down | checkpoints queue for eval; training continues; no promotion without a decision |
| gang cannot be placed | honest queue wait; backfill smaller jobs; do not scatter across racks |
| loss diverges | recipe-declared guard (NaN, loss spike) stops the run and keeps the last good checkpoint |
| RL reward hacking | verifier-first rewards; judge drift check; #3's anchor set |
| weight sync lags rollouts | staleness bound enforced per trajectory; report the distribution |

### The one worth dwelling on: the restart that trained twice on the same data

A run fails at step 4,100 and restarts from the checkpoint at 4,000. The
checkpoint has weights and optimizer state but the loader restarts at shard
0 — the data cursor was not in the checkpoint. The next 4,000 steps re-read
data the model already saw; the loss curve looks slightly *better* than
before, because the model has seen these examples. Evaluation regresses on
held-out tasks. Nothing alarms.

The fix is a field in the checkpoint, and the reason it is a "dwell on" is
that the failure is silent in every metric the training loop reports. Only
the lineage — *this candidate saw shards 0–40 twice* — can say what happened,
and only if the cursor was recorded.

### The second: the promoted model nobody can reproduce

A candidate is promoted. A month later a regression is suspected and the team
re-trains the tuple. The loss curve differs from step 200. The dataset version
was pinned; the code revision was pinned; the seed was pinned; the
*tokeniser* was not — a new tokeniser version changed the packing. Everything
in `Recipe` that can change the bytes the model sees has to be a digest, and
the tokeniser is the one people forget.

---

## 19. Cost controls

```text
LoRA by default for exploration; full fine-tune when the eval says it is needed
early stopping through #3 — returns gangs
asynchronous checkpoints — the tax is otherwise 5%
sequence packing — the cheapest throughput win
night-time gangs on the inference floor's slack
checkpoint retention by lineage, tiered otherwise
RL: adapters over a fixed base, so weight sync is megabytes
```

---

## 20. Tradeoffs

| decision | chosen | given up | choose otherwise when |
|---|---|---|---|
| FSDP over TP/PP at ≤30B | simplicity; node-local gathers | scaling past ~70B | larger models |
| gang scheduling | correctness for collectives | fragmentation, queue wait | jobs are single-GPU |
| async checkpoint | ~free | host memory per rank for the staging copy | memory is at the limit |
| candidate = tuple | reproducibility, attribution | naming convenience | never |
| eval in the loop | early stop, auto-promote | eval capacity per checkpoint | evals are very expensive |
| dedicated rollout fleet | no contention with users | more capacity | RL is rare and small |
| LoRA for RL policy | MB weight sync | some capability ceiling | full-parameter RL is required |
| retention by lineage | reproducibility | storage | never for promoted lineages |

---

## 21. Connections

The rollout loop is where this series closes on itself. An RL rollout is an
agent run (#1) in a sandbox (#4), graded by a verifier hierarchy (#3), served
by an inference engine (#2), and recorded with enough lineage to replay
(#5). The trajectories it produces are the same objects behavioural-variation
analysis consumes — and the same stochasticity applies: a rollout is one
sample, reward is a distribution over samples, and a policy that looks better
on one batch has not been shown to be better.

The forward direction is obvious from the diagram: the agent trajectories the
evaluation platform collects in production are a dataset, and the pipeline
from *observed failure* to *training signal* to *promoted model* is one
lineage graph across four of these designs.

---

## 22. Telling it in 45 minutes

```text
0–5     scope: post-training, not pre-training; the four services it calls
5–10    architecture; lineage as the spine; the registry is a graph
10–17   one SFT job; the memory arithmetic; DDP → FSDP; LoRA is a different profile
17–23   gang scheduling; topology; the shared pool and preemption to checkpoint
23–29   failure: what a checkpoint holds; async publish; stragglers
29–34   eval in the loop; candidate tuple; early stop returns GPUs
34–40   the RL loop: rollouts are agents; rollouts are the bottleneck; staleness; weight sync
40–45   capacity; the data-cursor story; build first
```

The opening mistake: starting from 4,096-GPU pre-training and Megatron.
Nobody asked, and the interesting problems here are the ones that appear at
64.

---

## 23. Questions the design has to survive

1. **Why can't a 27B fine-tune run on one 80 GB GPU?** ~16 bytes per
   parameter of state with mixed-precision Adam — 430 GB before activations.
   Shard it (FSDP) or freeze it (LoRA). §8.
2. **DDP versus FSDP — when?** DDP when the state fits per GPU and you want
   throughput; FSDP when it does not fit. At 27B full fine-tune it does not.
   §8.
3. **One GPU of 64 dies at hour nine.** Collective timeout → the gang is dead
   → restart on a new gang from the last checkpoint with weights, optimizer,
   cursor, RNG. Bounded by the interval; async checkpoints make the interval
   cheap. §10.
4. **How do you share GPUs with inference?** Inference keeps a floor;
   training takes the rest by priority; preemption only to a checkpoint;
   night-time gangs. §9.
5. **Twenty experiments — how do you pick one?** You don't, on Friday. Every
   checkpoint goes to #3; the orchestrator continues, stops early, or
   promotes. Early stop returns GPUs. §11.
6. **Reproduce a promoted model?** Same tuple → same data order, same
   trajectory within noise. Everything that changes bytes is a digest,
   including the tokeniser. Bitwise is not the bar. §11, §18.
7. **The RL loop is slow. Where?** Rollouts: generation costs more than
   training on it, ~2.5×. It is an inference problem; batch, cache, and size
   the rollout fleet accordingly. §12, §17.
8. **Rollouts arrive from an old policy.** Record `policy_version` per
   trajectory; the recipe declares the staleness bound. The field is the
   platform's job; the bound is the recipe's. §12.
9. **Reward hacking?** A reward model is a grader; verifiers first, judges
   pinned and calibrated, the same hierarchy as #3. §12.
10. **At 10×?** Gang fragmentation and checkpoint bandwidth before compute.
    Node-preserving placement, backfill, and host-local checkpoint staging.
    §9, §10.

---

## 24. What to build first

The **lineage**: candidate as a tuple, the registry as a graph, and a
checkpoint format that carries the data cursor and RNG. The scheduler can be
a queue and the trainer can be a script for months; a fleet of promoted
models whose provenance was not recorded cannot be fixed afterwards, and the
first regression nobody can reproduce is when that becomes clear.
