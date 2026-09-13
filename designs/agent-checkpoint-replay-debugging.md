# System Design #5 — Agent Checkpoint, Replay and Debugging Platform

> **Design a platform that can checkpoint, recover, replay, fork and debug
> long-running AI agents whose executions last hours, span model calls, tool
> calls, sandbox state and external side effects, and sometimes end wrong.**

The concrete case: a coding agent has been running for six hours. It has made
several hundred model calls, edited dozens of files, run tests, installed
packages, queried an external service, had three model requests retried by the
transport layer, and produced a final patch that is wrong. Another run of the
same task, yesterday, was right.

The questions are not only *can I resume it if the worker dies*. They are:

```text
What exactly happened?
What state was the agent in at step 143?
Why did this run fail when yesterday's succeeded?
Can I reproduce the failure?
Can I continue from step 143 with one thing changed?
Was it the model, the harness, a tool, the environment, or us?
```

**The framing that decides everything else:** recovery, replay and re-execution
are three different operations, and a system that conflates any two of them
is wrong about both.

```text
recovery        continue useful work after an infrastructure failure
replay          reconstruct what historically happened, from the record
re-execution    draw a new sample — a fresh continuation from a chosen state
```

A replay that calls the model again is not a replay; it is a re-execution
pretending to be one. A recovery that consumes recorded outputs is not a
recovery; it is a replay that will diverge from the world. Keeping the three
apart is most of the design.

It composes with the rest of the series:

```text
#1  Agent Execution Platform     owns live jobs, leases, sandboxes; emits events
#2  LLM Inference Platform       produces the model responses that get recorded
#3  Evaluation Platform          consumes forks and decides whether a
                                 difference is evidence
#5  this design                  records, reconstructs, resumes, replays,
                                 forks, compares, explains
```

The mechanics of a single checkpoint — manifest publish, fencing, the three
orderings, the lineage tree — are in
[Checkpoint and resume](../deep-dives/checkpoint-resume.md). This design
assumes them and builds the rest: the event model that makes replay possible,
the fork that makes causal debugging possible, and the comparison that turns
two runs into a finding.

---

## 1. Problem statement

A client can ask:

```text
checkpoint(run_id)                          materialise a recovery point now
resume(run_id, checkpoint_id?)              continue the live execution
replay(run_id, from_step, to_step)          reconstruct the record, no live calls
fork(run_id, checkpoint_id, overrides)      new run from a historical state
compare(run_a, run_b)                       where and how they diverged
inspect(run_id, step)                       the full state at a step
```

Agent state spans:

```text
conversation / context          model, prompt, harness versions
tool-call history               sandbox filesystem
pending and committed effects   environment configuration
external observations           budgets and step counters
```

Must support:

```text
runs of minutes to days
recovery from the latest committed checkpoint in under a minute
replay that reproduces recorded state hashes, or says where it cannot
forks at logical step boundaries, with lineage
comparison at several levels, not a transcript diff
30 days of full artifacts; metadata much longer
replay that never performs an external effect
```

---

## 2. Clarify before designing

| question | what it changes |
|---|---|
| Recovery only, or debugging too? | recovery needs a recent materialised state; debugging needs the full history and every output — the second is what this design is for |
| Are tools deterministic? | whether replay may re-run a tool or must consume its recorded output |
| Can agents cause external effects? | replay safety, an effect ledger, and the exactly-once boundary |
| Process-memory snapshots needed? | application-level checkpoint versus VM snapshot — and the two-tier answer |
| Recovery latency target? | cadence, and local versus remote snapshot storage |
| Must a run stay debuggable after the model and tools change? | pin by digest; retain artifacts; replay from outputs rather than re-execution |
| Fork from arbitrary steps, or only from checkpoints? | checkpoint cadence, or reconstruct-by-replay to reach any step |
| Who may replay? | a replay reveals everything the agent saw; access control is not optional |

**State the scope out loud:**

> I'll assume runs up to a day, recovery under a minute from a committed
> checkpoint, tools partly nondeterministic, external effects possible,
> fork at any committed step, and full artifacts retained for thirty days.

---

## 3. Non-functional requirements

### Recovery

```text
restore from latest committed checkpoint    < 60 s
work lost                                   ≤ one checkpoint interval
never resume into a state the history does not describe
```

The last line is the one that costs design effort. A fast resume into the
wrong state is worse than a slow one that fails loudly.

### Replay fidelity

```text
from checkpoint C, consuming recorded events through step t,
every recorded state hash is reproduced — or the first mismatch is reported
with a reason
```

Replay is deterministic **by construction**, because it consumes outputs. Any
divergence is therefore a finding about the record, not about the model.

### Safety

```text
replay performs no external effect, ever
fork performs effects only through the guarded effect layer, as a new run
```

### Storage and retention

```text
metadata and events     years
artifacts               30 days hot, then cold
physical snapshots      hours; local; never the durable record
```

### Debug latency

```text
inspect any step of any run within the window    seconds
first-divergence between two runs                seconds, from the index
```

---

## 4. Data model

Six entities. As before, the boundaries are the content.

### Run — one execution lineage

```text
Run {
  run_id
  task_ref
  candidate                model, prompt, harness, tools, inference config — by digest
  parent_run_id            null for a root
  fork_event_id            the parent step this run branched from
  base_checkpoint_id       what it was materialised from
  overrides                what differs from the parent
  status
}
```

A fork is a **new run** with lineage, never a mutation of the parent. The
parent's history is immutable from the moment it was written.

### Event — one committed transition

```text
Event {
  event_id, run_id, logical_step, seq
  kind                     MODEL_REQUESTED | MODEL_COMPLETED | TOOL_REQUESTED |
                           TOOL_COMPLETED | SANDBOX_MUTATION | EFFECT_INTENT |
                           EFFECT_COMMITTED | CHECKPOINT | RETRY | RECOVERY |
                           FORK | STEP_COMMITTED | RUN_COMPLETED
  attempt                  the same logical operation may have several
  input_ref, output_ref    content-addressed; never inline if large
  state_before, state_after   workspace and context fingerprints
  versions                 model, tool, environment — by digest
  timestamp
}
```

Two fields do most of the work. **`output_ref`** is what makes replay possible
at all: recording inputs reproduces nothing, because the model and the tools
are not functions. **`state_after`** is what makes replay *checkable*: it is
the assertion that replay verifies at every step.

### Artifact — large, immutable, content-addressed

```text
Artifact { artifact_id = hash(content), kind, size, tenant, retention_class }
```

Model responses, tool outputs, workspace diffs, test logs, snapshots. Shared
across checkpoints, forks and runs by reference.

### Checkpoint — materialised state plus a position

```text
Checkpoint {
  checkpoint_id, run_id, generation, checkpoint_seq, agent_step
  parent_checkpoint_id
  state_refs               context, tracked, untracked, tool state
  event_offset             the last committed event this state reflects
  effect_cursor            how far the effect ledger was drained
  versions
}
```

> A checkpoint is materialised state **plus** an event-log position **plus**
> lineage. The position is what ties a snapshot to the history that produced
> it. Without it, a snapshot says what existed, not why.

### Effect record — what crossed the boundary

```text
EffectRecord {
  operation_id             from the canonicalised intent, not from step
  run_id, generation
  class                    PURE | READ | IDEMPOTENT_WRITE | NON_IDEMPOTENT_WRITE
  status                   pending | committed | uncertain | reconciled
  result_ref
}
```

### Lineage — the run tree

Derived from `Run` and `Checkpoint`, indexed for the queries that matter:
*all continuations of this state*, *the live branch of this job*, *what this
fork changed*.

---

## 5. High-level architecture

```text
                         Agent Runtime (#1)
                               │  events, artifacts, cost
                               ▼
                     ┌────────────────────┐
                     │  Durable Event Log │   append-only, per-run order
                     └─────────┬──────────┘
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
   ┌────────────────┐  ┌───────────────┐  ┌───────────────┐
   │  Checkpoint    │  │   Artifact    │  │  Trace Index  │  derived,
   │  Coordinator   │  │    Store      │  │   Builder     │  rebuildable
   └───────┬────────┘  └───────┬───────┘  └───────┬───────┘
           ▼                   │                  ▼
   ┌────────────────┐          │          ┌───────────────┐
   │ Checkpoint     │◀─────────┘          │  Debug API    │
   │ Store          │                     │ inspect ·     │
   └───────┬────────┘                     │ compare ·     │
           │                              │ replay        │
           ▼                              └───────┬───────┘
   ┌────────────────┐                             │
   │ Restore Service│◀────────────────────────────┘
   └───────┬────────┘
     ┌─────┴──────┬──────────────┐
     ▼            ▼              ▼
  resume       replay          fork
  (live, #1)   (no network,    (new run, #1,
                no effects)     new lineage)
```

Four durable products, and the order to name them in:

```text
Event log       what happened, in causal order, with outputs
Artifacts       the large content those events point at
Checkpoints     materialised state at committed positions
Lineage         how runs, checkpoints, retries and forks relate
```

The trace index is *derived* — a search structure over the log — and can be
dropped and rebuilt. The event log is the truth.

---

## 6. Control plane vs data plane

| | control plane | data plane |
|---|---|---|
| owns | event log metadata, checkpoint coordination, lineage, restore decisions, effect ledger, index | live agent runs (#1), replay executors, fork runs (#1) |
| runs agent code | never | resume and fork do; replay does not |
| network | internal | resume/fork: sandboxed egress as #1; **replay: none** |

The replay executor is a data-plane component with an unusual property: it
executes the agent loop's *transitions* but never the agent's *calls*. No
model endpoint, no tool network, no effect service. It reads recorded outputs
and applies recorded mutations. That is enforced by placement — there is
nothing for it to reach — rather than by policy in the code, because the code
is the part that can be wrong.

A fork is a live run on #1 with a new run identity. It has the same rights as
any other run, no more; in particular its effects go through the guarded
effect layer and are recorded against *its* run, not the parent's.

---

## 7. Lifecycle — four paths

### Normal execution

```text
step k
  MODEL_REQUESTED → MODEL_COMPLETED (output recorded, hash of context after)
  TOOL_REQUESTED  → TOOL_COMPLETED  (output recorded, workspace hash after)
  EFFECT_INTENT   → EFFECT_COMMITTED (via the effect layer)
  STEP_COMMITTED  (event_offset advances; checkpoint eligible)
```

### Recovery

```text
lease expires (#1)  →  generation++  →  old worker fenced
  ↓
Restore Service: latest committed checkpoint on the live lineage
  ↓
materialise: physical snapshot if local, else logical checkpoint
  ↓
apply committed events after the checkpoint's offset
  ↓
reconcile the effect ledger: pending → search / declare / park
  ↓
new worker resumes at the next uncommitted step
```

### Replay

```text
checkpoint C  →  events [C.offset, t]  →  inject outputs  →  verify hashes
                                                      ↓
                                     match          mismatch → REPLAY_DIVERGENCE
```

### Fork

```text
choose (run, checkpoint or step)  →  materialise state  →  apply overrides
  ↓
new run_id with parent_run_id, fork_event_id, base_checkpoint_id, overrides
  ↓
runs on #1 as a fresh job; records its own events
```

---

## 8. Deep dive — the event model, and what must be recorded

The first rule, and the one most systems get wrong by recording too little:

> **Record outputs, not just inputs.**

Inputs let you re-issue a call. They do not let you reproduce its result,
because the model is a sample and the tools read a world that moves. Replay
needs every model response and every tool output, by reference, at the moment
they were produced. Measured on this workload: forked continuations from an
identically reconstructed state reproduced the next three actions in 14 of 16
cases and the final state in 0 of 15. Re-issuing calls reconstructs nothing
past a few steps.

### Per event

```text
what was asked         input_ref
what came back         output_ref
which attempt          attempt, retry_reason
what the state was     state_before, state_after   (context hash, tracked hash, workspace hash)
under which versions   model, tool, environment digests
```

The `attempt` field matters more than it looks. Transport failures re-sample
the model: one batch measured 333 model calls, 350 attempts, one step sampled
six times. Without `attempt`, replay cannot know which response the agent
actually acted on, and comparison cannot tell a model difference from a
transport one.

### Three fingerprints, not one

```text
context hash        the transcript as the model will see it next
tracked hash        canonical diff of tracked source
workspace hash      tracked + untracked + tool state
```

They diverge in different situations. Two runs with the same tracked hash and
different workspace hashes are the case from the checkpoint deep dive — same
code, different scratch files — and only the third fingerprint sees it.

### Recorded out of band

Instrumentation must not enter the agent's own context (#1 §13). The runtime
records; the agent does not observe itself being recorded. Otherwise the
measurement changes the trajectory it is measuring.

### Size and sampling

Every event's *digest* is kept. Every event's *payload* is kept for the
retention window **if recovery or replay is expected to serve it**. Sampling
payloads to save storage silently removes the entry replay was going to need —
and it fails at step 12 of a 140-step run with a divergence that looks like
nondeterminism and is actually a hole in the record. If a payload class is
sampled, replay must report *unrecorded*, never *diverged*.

---

## 9. Deep dive — checkpoints as committed prefixes

> Full mechanics: [Checkpoint and resume](../deep-dives/checkpoint-resume.md) —
> manifest publish, fencing, generation/seq/step, the lineage tree,
> `recovery_head`.

What this design adds is the connection to the log.

### The commit boundary

```text
model says: delete file A
sandbox deletes A
  ← a snapshot here contains the deletion; the log does not
tool result committed
STEP_COMMITTED
  ← a checkpoint here is consistent
```

A checkpoint taken between the mutation and its commit resumes into a state
the history does not describe: the runtime either repeats the deletion or
believes it never happened. So checkpoints are eligible only at
`STEP_COMMITTED`, and the manifest carries the `event_offset` of that commit.

> **Every published checkpoint corresponds to a committed prefix of the event
> log.** The offset is part of the manifest, and the manifest is what gets
> fenced.

### Two tiers

| | logical checkpoint | physical snapshot |
|---|---|---|
| contains | context, tracked diff, untracked delta, tool state, offset, budgets | VM memory + disk |
| size | KB to a few MB | GB |
| cadence | every committed step — cheap, because most steps change nothing | every N minutes, or before an expensive milestone |
| storage | durable object store, content-addressed | local to the host, short-lived |
| serves | cross-host recovery, replay, fork | same-host restart in seconds |
| portable | yes | across kernel and runtime versions, no |

Recovery composes them: latest local snapshot if the host survived, else the
latest durable logical checkpoint, then apply committed events past its
offset. The physical tier is an optimisation of restore latency and never the
durable record — a VM snapshot restores a socket the far end closed an hour
ago and a credential that expired.

### Adaptive cadence

```text
checkpoint before     an external effect, a large mutation, an expensive tool,
                      context compaction
checkpoint less       during runs of cheap deterministic steps
```

The event-triggered half is what makes the effect story tractable: if a
checkpoint always precedes an effect, the recovery boundary and the effect
boundary coincide.

---

## 10. Deep dive — replay

Replay is the operation that turns the record into something checkable.

```text
start from checkpoint C
for each event in [C.offset, target]:
    MODEL_COMPLETED      inject recorded response; do not call the model
    TOOL_COMPLETED       inject recorded output;   do not run the tool
    SANDBOX_MUTATION     apply the recorded delta
    EFFECT_*             consume the recorded result; never execute
    STEP_COMMITTED       compute state hashes; compare with state_after
```

### The invariant

> From C, consuming committed events through t reproduces every recorded
> `state_after`. The first mismatch is `REPLAY_DIVERGENCE` at that step, with
> a reason.

Because replay never samples anything, a divergence is never "the model did
something else". It is one of:

| reason | meaning | fix |
|---|---|---|
| `unrecorded` | a payload was sampled or expired | retention, not replay |
| `nondeterministic_apply` | the recorded mutation does not reproduce the hash — timestamps, ordering | canonicalise the mutation or record the resulting state |
| `environment_drift` | base image or fixture differs from the recorded digest | pin and retain |
| `record_inconsistent` | events out of order, missing commit | a runtime bug, and the most valuable finding |

Replay is therefore also **the test of the recording**. Run it continuously on
a sample of completed runs, the way a backup is restored to prove it exists.
A replay that has never been run is a record that has never been checked.

### What replay cannot do

Background processes the agent started, wall-clock-dependent behaviour, and
in-flight calls at a crash are not in the record. Replay reports them as
boundaries, not as failures, and the runtime declares up front what it does
not preserve (deep dive §8).

### Replay is not reproduction

Replay reconstructs the record. It does not tell you what the model would do
now — that is a fork, and it will differ. Say it every time the word comes up:
*replay from recorded outputs, yes; reproduce by re-execution, no.*

---

## 11. Deep dive — fork and causal debugging

Replay says *what happened*. It cannot say *why*, because a single history has
no counterfactual. The fork is the counterfactual.

```text
Run A     S0 → S1 → S2 → S3 → S4 → FAIL
                          │
                          └── fork at S3, override {prompt: v2}
                                    │
                                    ▼
                                   S3' → … → PASS ?
```

The child inherits everything through `S3` — environment snapshot, tracked
and untracked workspace, transcript, tool observations — and overrides one
set of things:

```text
model · prompt · harness · tool policy · inference config · an injected message
```

It gets a new `run_id`, with `parent_run_id`, `fork_event_id`,
`base_checkpoint_id` and `overrides` recorded. It runs on #1 as an ordinary
job. Its effects are its own.

### Materialising S3 correctly

The fork is only meaningful if the child really starts from `S3`. Restore the
transcript and the tracked diff but drop the scratch files and the child is a
different agent that happens to share source. So materialisation is
**verified**: rebuild the state in a fresh sandbox, recompute all three
fingerprints, compare with the recorded `state_after` at `S3`. A mismatch is a
refused fork, not a warning.

### One override at a time

```text
same S3, different model        →  model effect
same S3, different prompt       →  prompt effect
same S3, injected intervention  →  is the failure recoverable from here?
same S3, 16 continuations       →  outcome variance from this state
```

The last row is the one that keeps the others honest. Before attributing a
difference to an override, know how much the *unchanged* continuation varies.
Measured: from one reconstructed state, 0 of 15 unmodified continuations
reached the donor's final state. Against that baseline, a single forked run
that passes proves nothing; a distribution that shifts does.

Which is the hand-off to #3: **#5 produces controlled branches; #3 decides
whether the difference across branches is evidence.** The lineage fields are
what let #3 compute an effective N — sixteen continuations of one state are
sixteen samples of *that state*, not sixteen independent runs.

### Copy-on-write

A fork must not copy six hours of artifacts. Content addressing makes the
child's references to the parent's blobs free; only what the child changes is
new. A fork tree of a hundred branches from one state costs one state plus a
hundred deltas.

---

## 12. Deep dive — comparing two runs

The naive debugger diffs transcripts. Two runs can differ in every token and be
doing the same thing, and agree on every token until the one that matters.

### Fingerprints at several levels

```text
L0   event kind                 model call, tool call, effect
L1   action identity            which tool; command signature (sed -n, pytest)
L2   normalised arguments       the exact command
L3   canonical workspace        tracked hash; workspace hash
L4   verifier-relevant state    does the target test pass; is the target file touched
L5   outcome
```

Two runs are aligned level by level, and the comparison reports **the first
divergence at each level**:

```text
first action divergence         step 12   (sed on different line ranges)
first state divergence          step 31   (workspace hash)
first verifier-relevant         step 88   (target test first fails only in A)
outcome                         A FAIL, B PASS
```

That table is the debugger's starting point, and it is deliberately not a
claim about cause. Runs diverge early and harmlessly, reconverge, and part
again — measured trajectories show exactly that topology. The first token
difference is almost never the consequential one.

### Harmless versus consequential

A divergence is *consequential* if it reaches the outcome. The comparison
cannot establish that from two runs. It can establish it from many: the
platform's own forks, or #3's repeated trials, aggregated over the level-L3
and L4 fingerprints. That is what turns *where did they differ* into *which
difference mattered* — and it is analysis over recorded state, not a new
recording. The platform's job is to make the fingerprints exist at every step
so the analysis is possible later.

### Layer attribution, from the record alone

Some of the *why* is answerable without forking, because the events carry it:

| layer | visible in the record as |
|---|---|
| infrastructure | RETRY and RECOVERY events; censoring; attempt > 1 |
| inference | MODEL_COMPLETED with a different serving digest; attempt > 1 on a step |
| environment | environment digest differs; tool output differs on identical input |
| tool | TOOL_COMPLETED differs for identical L2 arguments under the same environment |
| harness | the same model output produced a different action |
| model | identical context hash, different model output |

Only the last two need a fork to separate; the others are a query.

---

## 13. External effects, and why replay must be inert

> Full treatment: [Scheduler, lease, recovery](../deep-dives/scheduler-lease-recovery.md) §8, §20.

Every tool call carries an effect class. Replay honours it absolutely:

```text
PURE, READ              consume the recorded output (re-running is allowed
                        only in a fork, and even then it is a new observation)
IDEMPOTENT_WRITE        never executed in replay; recorded result returned
NON_IDEMPOTENT_WRITE    never executed in replay; recorded result returned
```

The replay executor cannot reach the effect service. That is the enforcement.

For **recovery**, the effect ledger is what the restore path reconciles:

```text
pending after restore   →  search the provider for operation_id
                           found → committed; not found → execute with the key
                           no read path → tool's declared preference:
                                 at_least_once → resend
                                 at_most_once  → park for a human
```

For **forks**, effects are real — the child is a live run — and that is why
forking a run that sent emails is a decision, not a default. The platform
lets an experiment declare that its forks run with the effect layer in
*record-only* mode: intents are logged, nothing is sent, and the agent
receives a synthetic success. That is the right setting for debugging and the
wrong one for measuring production behaviour; the experiment says which.

---

## 14. Storage

Four stores, because four access patterns:

| store | holds | pattern |
|---|---|---|
| metadata DB | runs, checkpoints, lineage, effect ledger | small rows, transactional, partition by run |
| event log | ordered events per run | append; range scan by (run, offset) |
| object store | artifacts, snapshots — content-addressed | write once, read by hash, tiered |
| trace index | derived: fingerprints by step, divergence pairs, tool/result lookups | query; rebuildable from the log |

Content addressing does three jobs at once: dedup across steps (most steps
change no workspace bytes), integrity (the hash is the check), and free forks
(references, not copies). The one thing it complicates is deletion — a job's
blobs may be shared by another job — and tenant isolation, which is why the
address is namespaced by tenant.

Delta chains get compacted when restore latency, not storage, says so: a
periodic full state every N steps bounds the number of deltas any restore or
replay has to apply.

---

## 15. Observability

Platform:

```text
checkpoint latency, failure rate       restore latency p50/p95/p99
recovery success rate                  work recomputed after recovery
event-log lag from the runtime         replay divergence rate, by reason
snapshot bytes per run                 index freshness
```

Debugging:

```text
first divergent step per level         attempt > 1 per step
effect ambiguity count                 fork outcome distribution per state
```

And one that must exist and usually does not:

```text
recovery correctness failures      resumed, then contradicted its own history
```

detected by the fingerprint rebuild on a sample of restores. A system that
resumes quickly into the wrong state generates confused agents, not alerts,
unless this is measured.

---

## 16. Security and multi-tenancy

A checkpoint is a complete copy of a workspace that ran untrusted code, and a
replay reveals everything the agent saw.

```text
scrub credentials at write; store references, re-mint on resume
encrypt at rest, key per tenant; namespace content addresses by tenant
replay and inspect are privileged operations with their own audit log
forks inherit the parent's tenant and quota
checkpoints are user data: deletion requests apply, and refcounts decide
```

The dedup-versus-isolation tension is real: identical content across tenants
would dedupe, and that is an information leak. Namespace first, dedupe within.

---

## 17. Capacity estimation

From #1's scale: 100K jobs/day, ~40 steps, 30 min median.

### Events and metadata

```text
100K × 40                =  4M events/day
× ~1 KB                  =  4 GB/day metadata          trivial
```

### Artifacts

```text
~25 KB per step, compressed (command, output, diff)
4M × 25 KB               =  100 GB/day
30 days hot              =  ~3 TB                       fine
```

Consistent with #1 §17's 1 MB/job. Model responses dominate; workspace deltas
are small because ~75% of steps change no tracked bytes.

### Physical snapshots — the number that changes the shape

```text
microVM memory + disk delta   ~1 GB compressed
every 10 min, 30 min median   ~3 per job
100K jobs/day                 ~300 TB/day
```

**Not storable.** So physical snapshots are local, short-lived, and never the
durable record; the durable tier is the logical checkpoint at KB–MB. The
two-tier design in §9 is forced by this arithmetic, not chosen for elegance.

### Replay and fork load

```text
replay    CPU only, no GPU — reads ~1 MB per run; cheap; run continuously on a sample
fork      a live #1 job; costs what a run costs; K forks of one state = K runs
```

Forks are the expensive product and they are #1's capacity, not this
system's. A debugging session that forks sixteen continuations of a
three-hour run has scheduled forty-eight agent-hours.

### Index

```text
4M events/day × fingerprints at 6 levels   →  a few hundred GB/year
```

Derived, partitioned by run, rebuildable.

---

## 18. Failure modes

| failure | handling |
|---|---|
| worker dies during checkpoint | unpublished manifest is invisible; restore the previous committed one |
| blobs durable, manifest commit fails | orphans collected later; no checkpoint exists |
| manifest before blobs | forbidden by the publish protocol |
| event written twice | idempotent on (run, step, seq, attempt) |
| old worker writes after recovery | generation fencing on the log and the manifest |
| model call retried by transport | every attempt recorded; the acted-on one is marked |
| external write outcome unknown | ledger `uncertain`; reconcile; never blind retry |
| artifact expired | replay reports `unrecorded` at that step; recovery unaffected if past the checkpoint |
| snapshot corrupt | digest fails; previous checkpoint |
| replay reaches an effect | structurally impossible; the executor has no route |
| fork mutates parent | impossible; parent artifacts immutable, child is a new run |
| model version gone | replay still works; fork under the same pins does not, and says so |

### The one worth dwelling on: divergence at step 12, bug at step 140

A run fails at step 140. Replay diverges at step 12 and stops. The engineer
concludes the record is broken and gives up.

Step 12 was a `grep` whose output exceeded the payload cap and was
**sampled**: the digest was kept, the bytes were not. Replay cannot inject
what it does not have, computes a different context hash, and correctly
reports a mismatch. Nothing about step 12 matters to the failure; the record
simply has a hole, and the hole is upstream of everything interesting.

Two things fix this, and neither is "record everything":

- **Report the reason.** `unrecorded` is not `diverged`. Replay continues
  past an unrecorded step in *degraded* mode — inject the digest as a
  placeholder, mark every later hash as unverifiable-for-context — so the
  workspace fingerprints, which are intact, still reach step 140.
- **Retention follows use.** If replay is expected to serve a payload, its
  retention is a correctness property. Sample observability-only payloads;
  never sample the ones replay reads.

### The second: resumed quickly into the wrong state

A restore applies the tracked diff and the transcript and forgets the
untracked delta. The agent resumes in twelve seconds, reads `repro.py` back,
and is told it does not exist. It spends the next forty steps and eleven
dollars debugging a file system. The restore latency dashboard is green.

Only the workspace fingerprint rebuild on restore catches it — the tracked
hash matches. This is why recovery correctness is a metric and not an
assumption.

### What breaks first at 10×

Not the event log. The large mutable state — snapshots, images, tool artifacts
— and then the index. Incremental snapshots, cross-run dedup, tiered storage,
and an index that is derived and can lag.

---

## 19. Cost controls

```text
logical checkpoints every step; physical rarely and locally
content addressing — most steps store nothing new
retention by use class: recovery-critical never sampled; observability tiered
forks are budgeted as runs, through #1's accounting, against the experiment
replay is cheap; run it continuously on a sample as the record's own test
```

---

## 20. Tradeoffs

| decision | chosen | given up | choose otherwise when |
|---|---|---|---|
| record outputs, not just inputs | true replay | storage; retention policy work | never — inputs reproduce nothing |
| replay inert by placement | cannot perform an effect | cannot re-run a pure tool "just to check" | never; that is a fork |
| two-tier checkpoints | fast local restore + durable portable state | two mechanisms | short jobs: restart is cheaper than either |
| fork = new run | immutable history; clean lineage | in-place "edit and continue" | never |
| verified materialisation | no confused agents | seconds per restore | never; it is the correctness check |
| content-addressed artifacts | dedup, integrity, free forks | deletion and tenant isolation are harder | tiny scale |
| derived index | rebuildable; log stays simple | staleness | queries must be transactional (rare) |
| fingerprints at six levels | comparison that is not a token diff | computing them at every step | agents without a workspace notion |

---

## 21. Connections

This is the platform that behavioural-variation analysis runs on. The
pipeline it enables:

```text
repeated runs of one task              (#3, or forks from one state here)
      ↓
fingerprints per step at several levels
      ↓
where trajectories diverge, and whether the divergence reaches the outcome
      ↓
fork with one override, from the last shared state
      ↓
does the outcome distribution move?
```

Two measured facts shaped the design. Trajectories diverge early, reconverge,
and part again — so the comparison reports divergence per level, not a single
first difference. And unmodified continuations from one state rarely reach
the same end — so a fork is evidence only against the variance of the
unmodified branch, which is why fork and evaluation are two systems that
share lineage rather than one.

The forward direction: once forks are cheap and verified, the same machinery
runs *before* failure — risk rises, checkpoint, fork alternatives, continue
the best branch. The debugger becomes a steering mechanism.

---

## 22. Telling it in 45 minutes

```text
0–5     the three semantics; scope; what #1 and the deep dive already own
5–10    architecture: log, artifacts, checkpoints, lineage; index is derived
10–17   event model: outputs not inputs; attempt; three fingerprints
17–23   checkpoint = state + offset + lineage; commit boundary; two tiers
23–29   replay: injection, hash check, divergence reasons; replay tests the record
29–35   fork: verified materialisation; one override; variance baseline; hand-off to #3
35–39   compare: six levels, first divergence per level, harmless vs consequential
39–42   effects: replay inert; recovery reconciles; forks declare a mode
42–45   the snapshot arithmetic; the step-12 story; build first
```

The opening mistake: starting with VM snapshots. They are the least
interesting tier and the one that does not scale.

---

## 23. Questions the design has to survive

1. **Isn't a VM snapshot enough?** It restores memory, not meaning: sockets
   the far end closed, credentials that expired, and no connection to the
   history. Use it as the fast local tier; the durable record is logical.
   §9, §17.
2. **How is replay different from running it again?** Replay consumes
   recorded outputs and calls nothing; it reproduces the record and verifies
   hashes. Running again is a fork: a new sample, new lineage. §10, §11.
3. **Replay diverged at step 12. Now what?** Read the reason. `unrecorded`
   means retention, not the model; continue degraded on the intact
   fingerprints. `record_inconsistent` is a runtime bug and the best finding
   of the day. §10, §18.
4. **The run sent an email at step 50. Replay it?** Replay cannot send
   anything — the executor has no route. A fork past step 50 is a live run
   and either goes through the effect layer or runs record-only, declared by
   the experiment. §13.
5. **How do you know a restore was correct?** Rebuild in a fresh sandbox,
   recompute all three fingerprints, compare with the recorded state. Tracked
   hash alone passes the broken case. §11, §18.
6. **Fork at step 143, change the prompt, it passes. Was it the prompt?** Not
   from one run. Sixteen unmodified continuations of that state give the
   baseline; measured, most reach different ends. A shifted distribution is
   evidence; one pass is a sample. §11.
7. **Which run's history does a fork's effect belong to?** The child's. Parent
   history is immutable from the moment it is written; a fork is a new run
   with lineage, never an edit. §4, §11.
8. **Where does the storage go at scale, and what do you cut?** Physical
   snapshots — hundreds of TB/day if durable, so they are not. Never cut
   recorded outputs replay serves; sample observability-only payloads. §17,
   §8.
9. **Two runs differ from token one. How do you find the difference that
   mattered?** First divergence per level — action, arguments, workspace,
   verifier-relevant, outcome — then many runs, not two, to see which
   divergences reach the outcome. §12.
10. **At 100×?** Large mutable state first — incremental snapshots, cross-run
    dedup, tiering — then the index, which is derived and may lag. The log
    itself partitions by run and is not the problem. §18.

---

## 24. What to build first

The **event log with recorded outputs and per-step fingerprints**, and the
**lineage fields** on runs and checkpoints. Recording cannot be added after
the fact: a run that was not recorded with its outputs can never be replayed,
and a fork whose parent was not fingerprinted can never be verified. Replay,
compare and fork are all *readers* of that log and can arrive in any order;
the log is the thing that, once ten thousand runs have been executed without
it, has cost ten thousand runs of debugging you cannot do.
