# Deep dive — Checkpoint and Resume

> Belongs to [System Design #1](../designs/agent-execution-platform.md), §12.
> Pairs with [Scheduler, lease, failure recovery](scheduler-lease-recovery.md):
> the lease decides **how fast a failure is noticed**, the checkpoint decides
> **how much is lost** when it is.

**What is being tested.** Whether you understand that a checkpoint is a
*consistency boundary*, not a backup. Most candidates describe saving files.
The interesting questions are all about what makes a saved state legitimate to
resume from.

---

## 1. What a checkpoint is

The common mistake:

> "Save the current workspace."

Sometimes enough for a batch job. Not enough for a long-running agent. The
useful definition:

> **A checkpoint is a recoverable boundary in the agent execution state
> machine.**

It must hold enough that after a crash, preemption or migration the system can
resume at a point that is **semantically unambiguous**.

```text
checkpoint ≠ workspace snapshot
```

```text
Checkpoint
├── agent conversation / context
├── tracked source state
├── untracked workspace
├── tool / runtime metadata
├── execution step
├── token / time / cost budget
├── external-effect state
└── model / session configuration
```

The one that is learned rather than read:

> **Restoring the source code is not restoring the agent.**

Apply only the tracked diff and the source fingerprint matches while the scratch
files and the message history do not. What comes back is a different execution
state that happens to have the same source. The agent remembers writing a
reproduction script, reads it back, and is told the file does not exist.

---

## 2. Resume is not restart-from-state

Worth volunteering, because it shows you have thought about semantics rather
than mechanics.

An agent crashes at step 50. Restore

```text
repo diff @ step 50
```

and start a fresh agent. That is **restart from source state**. It is a
perfectly reasonable product behaviour and it is not the same as

> **resume the original execution**

which additionally needs conversation history, scratch files, tool results,
current budgets and execution metadata.

And even with all of those restored, the next model call may take the trajectory
somewhere else. So a checkpoint has to declare **which semantics it offers**:

| | restores | promises |
|---|---|---|
| restart from state | the repository | a fresh attempt on the same code |
| resume | repository + context + budgets | continuation of the same execution |
| replay | everything, plus recorded outputs | the same trajectory — see §12 |

Do not use these three words interchangeably in an interview. The follow-up
*"so is it exactly the same run?"* is coming, and §12 is the answer.

---

## 3. Schema, and where things live

```text
AgentCheckpoint {
  checkpoint_id
  job_id
  execution_id
  generation                  the fencing generation that wrote it
  step

  agent_context_ref
  tracked_workspace_ref
  untracked_workspace_ref
  tool_state_ref

  external_effect_cursor      how far the effect log was drained
  token_budget_remaining
  wall_time_budget_remaining
  cost_budget_remaining

  model_config
  agent_config
  created_at
  checksum
}
```

Large objects do not go in the database:

```text
metadata      → transactional store
large blobs   → object store

message history      → compressed blob
workspace tar/delta  → object store
checkpoint manifest  → database
```

Two fields worth defending because they are easy to leave out:

**`generation`** ties the checkpoint to the lease that produced it. A stale
worker's checkpoint is rejected by generation, exactly as in the fencing
argument — the checkpoint store is one of the stores that does the rejecting.

**`external_effect_cursor`** records how far the durable effect log was drained.
Without it, resume cannot tell which irreversible actions already crossed the
boundary, and the whole §21 machinery of the previous deep dive has nothing to
read.

---

## 4. The hard problem: a checkpoint must be consistent

Say a checkpoint is three objects:

```text
messages.json
tracked.diff
untracked.tar
```

and the write order is

```text
write tracked.diff     ✓
write messages.json    ✓
write untracked.tar    ← crash
```

If recovery accepts the first two as a checkpoint, it resumes an **internally
inconsistent state**: a transcript that describes files the workspace does not
contain.

**This is worse than having no checkpoint at all**, because it looks legitimate.
A missing checkpoint costs work; a half-written one costs trust, and the failure
surfaces much later as agent behaviour nobody can explain.

So a checkpoint cannot be "write several files and hope". It needs a commit
boundary.

---

## 5. Manifest publish: data first, pointer last

Write every part as an **immutable blob**:

```text
blob A = messages
blob B = tracked workspace
blob C = untracked workspace
blob D = tool state
```

Only when all of them have landed, atomically publish a small manifest:

```text
CheckpointManifest {
  checkpoint_id
  messages_blob  = A
  tracked_blob   = B
  untracked_blob = C
  tool_blob      = D
  step   = 50
  status = COMMITTED
}
```

Recovery honours **committed manifests only** and never looks at loose blobs.

```text
write blobs
    ↓
all succeed?
    ↓
publish manifest atomically
```

Crash before the publish leaves orphan blobs, collected later (§10). Crash after
it leaves a complete checkpoint. **There is no state in which half a checkpoint
is visible.**

This is not distributed two-phase commit. It is the much cheaper pattern that
covers most of the same ground:

> **data first, pointer last** — make the only mutable thing a single small
> atomic write.

The same shape appears in filesystem journals, in Git (objects then ref), and in
object-store table formats. Naming that lineage is a cheap credibility signal.

---

## 6. Content-addressed blobs, and what deduplication actually buys

Name every blob by the hash of its contents. Then identical content is stored
once, no matter how many checkpoints, attempts or jobs reference it.

This matters because agent state is **highly repetitive**:

- Between consecutive steps the workspace usually does not change at all — most
  actions are reads. In one measured trajectory the tracked source moved on 12
  of 52 steps; the other 40 produced a byte-identical workspace.
- Retries of the same job re-traverse much of the same state.
- Jobs on the same repository share the same base tree.

So content addressing turns "checkpoint every step" from prohibitive into
routine: 40 of 52 checkpoints store *no new workspace bytes at all*, only a
manifest that points at an existing blob.

### The transcript is the part that actually grows

The workspace is repetitive; the conversation is monotonic. It only ever gets
longer, reaching tens of thousands of tokens by the end of a run.

Snapshotting it per checkpoint is quadratic:

```text
40 checkpoints × average 100 KB  ≈ 4 MB per job, for one transcript
```

Store it as an **append-only log** instead and the checkpoint references an
offset:

```text
transcript log, appended per step   ≈ 200 KB total
checkpoint holds (log_id, offset)
```

A 20× reduction, and it falls out of the observation that **the transcript is
append-only and the workspace is not.** Different shapes want different storage;
using one mechanism for both is what makes checkpointing look expensive.

---

## 7. Incremental checkpoints and delta chains

Full snapshots are simple and large. Deltas are small and chain:

```text
C0 full
C1 = C0 + Δ1
C2 = C1 + Δ2
...
C9 = C8 + Δ9        ← restoring C9 replays nine deltas
```

Restore cost grows with the chain, and **the chain is exactly what you depend on
during an incident**, when everything else is already going badly.

| | full snapshot | incremental |
|---|---|---|
| restore | simple, self-contained | replay the chain |
| write | amplified | fast |
| storage | large | small |
| latency | high | low |
| GC | trivial | must not break a chain |

Standard resolution: **periodic full plus deltas between them.**

```text
full  every 50 steps
delta every  5 steps
```

Restore is then bounded at ten applications. Choose N by restore latency
budget, not by storage: the storage saving is already captured by content
addressing.

With content-addressed blobs the distinction softens — an unchanged workspace is
the same blob, so the "delta" is free and the chain is short in practice. The
delta machinery is only earning its complexity for large workspaces that change
a little on every step.

---

## 8. What cannot be checkpointed

Being able to list this is a stronger signal than describing what can.

| state | checkpointable? | approach |
|---|---|---|
| files, transcript, budgets | yes | as above |
| environment variables set by the agent | yes | part of tool state |
| **background processes** | no | record intent, restart on resume, or forbid them |
| **open file handles, sockets** | no | re-derive; the resumed agent reopens |
| **installed packages** | technically yes | far cheaper to re-derive from the image digest |
| **external service state** | no | the effect cursor (§3) and reconciliation |
| **in-flight model call** | no | checkpoint boundaries are between steps (§9) |

The rule that keeps checkpoints small: **do not store what can be re-derived.**
An image digest plus a lockfile reproduces the installed packages; storing the
whole filesystem to capture them multiplies checkpoint size by a thousand for
nothing.

The one that trips people: a background process the agent started —
`python server.py &` — is genuinely lost. Either declare that the platform does
not preserve them and expose that to the agent's prompt, or treat starting one
as an effect that gets replayed on resume. Silently losing it produces an agent
that believes a server is running and is confused for the rest of the run.

---

## 9. When to take one

Checkpoints go **at step boundaries only**, where the agent's state is coherent:
the last message is an observation, not a half-finished action. There is no
meaningful checkpoint in the middle of a tool call, which is why a tool
interrupted by a crash is re-run from the previous boundary and why tool
idempotency carries that weight.

Frequency is a tradeoff to state, not a number to memorise:

With interval `T` and failures arriving at random, a crash lands on average
halfway through an interval:

```text
expected lost work  ≈ T/2
checkpoint overhead ∝ 1/T
```

One term rises with `T`, the other falls, so a balance exists. **Do not derive
the optimum in an interview** — stating the shape of the tradeoff is the answer;
producing a formula is a distraction from the part that is actually judged.

With content addressing the second term is much smaller than it looks, which
pushes the answer toward *frequently*.

**Periodic plus event-triggered:**

```text
every N steps
AND immediately before:
    an external side effect       ← makes the recovery boundary and the
    a large code modification        idempotency boundary the same boundary
    an expensive tool call
    context compaction            ← the transcript is about to stop being
                                     append-only
```

Context compaction deserves the callout: when the agent summarises and discards
history, the append-only assumption of §6 breaks. Checkpoint before it, and
treat the compacted transcript as a new log rather than an append to the old one.

---

## 10. Checkpointing does not make side effects transactional

The limit of everything above, and worth stating before an interviewer finds it.

```text
checkpoint C
     ↓
send email succeeds
     ↓
crash before recording success
```

Restore C. The system still does not know whether the email went out.

A checkpoint answers **where do I resume from**. It says nothing about effects
that already crossed the system boundary, which remain the province of
write-ahead intent, idempotency, reconciliation and a guarded effect service
([previous deep dive, §20](scheduler-lease-recovery.md)).

> **Checkpointing recovers computation state; it does not make external side
> effects transactional.**

Keeping these two mechanisms distinct is the point. Checkpointing before a side
effect (§9) narrows the window; it does not close it, and a design that claims
otherwise has not understood either half.

---

## 11. Tool state: what actually has to be saved

It depends entirely on the tool, and the split is clean.

**Stateless reads** — `grep`, `cat`, search. Nothing internal to preserve; the
result is already in the transcript.

**Long-running processes** — a `pytest` run, a build, a browser session. Two
options:

| | approach | cost |
|---|---|---|
| A | checkpoint the process itself — CRIU, VM memory snapshot | complex, fragile, large |
| B | checkpoint the *logical* state and re-run the tool | usually far more practical |

Option B in practice: store the command, its inputs, the working directory, and
re-execute on resume.

```text
pytest command
inputs
working directory
      ↓ on resume
re-run
```

Re-running a test suite costs minutes. Freezing and thawing a Linux process
correctly costs a subsystem. **Choose B unless the tool is genuinely
irreproducible**, and notice that B is only safe because the tool is in the
"pure read" class — re-running a deploy is a different conversation.

---

## 12. Could a VM snapshot replace all of this?

A likely challenge, especially once Firecracker is on the whiteboard: freeze
memory and VM state and the problem disappears.

**What it buys**

```text
fast restore
the application needs no understanding of its own state
processes survive
```

**What it does not**

```text
snapshots are large
portability across host kernel / runtime versions
network connections cannot be frozen — the world outside moved on
credentials inside the image go stale
external side effects are still not addressed
```

The last two matter most: a VM snapshot restores a socket that the far end
closed twenty minutes ago, and a token that expired. The agent resumes into a
world that no longer matches its memory.

So a mature design uses **both, at different tiers**:

```text
fast local recovery        →  VM snapshot
durable cross-host recovery →  logical application checkpoint
```

Same-host restart after a process crash takes the snapshot and is back in
seconds. Migration to another host, another zone, or three hours later takes the
logical checkpoint, which is portable and small. Presenting them as competing
options is the weaker answer; presenting them as tiers is the real one.

---

## 13. Checkpoints go through fencing too

A checkpoint commit is a durable write, so it obeys the rule from the previous
deep dive: **it carries the execution generation, and the store enforces it.**

```text
worker A  generation 7
worker B  generation 8

commit_checkpoint(job=123, generation=7)
store sees current generation = 8
      → reject
```

Without this, a partitioned worker A finishing a slow upload can publish a
checkpoint that becomes the "latest" and overwrites B's recovery point with
older state — a live worker losing work to a dead one.

### And *latest* cannot mean *newest timestamp*

The classic trap:

```sql
SELECT checkpoint ORDER BY created_at DESC LIMIT 1     -- wrong
```

Clock skew, a stale worker writing late, and network delay all break it. Order
by **ownership first, sequence second**:

```text
(generation = 8, checkpoint_seq = 12)
     is newer than
(generation = 7, checkpoint_seq = 100)
```

A checkpoint from a revoked generation is never newer than one from the current
generation, whatever its sequence number or its clock says. **Ownership
dominates recency** — the same principle as fencing, applied to selection rather
than to admission.

---

## 14. Garbage collection

Content addressing plus manifest publish creates two kinds of garbage:

- **Orphan blobs** — written before a crash that prevented publication.
- **Unreferenced blobs** — the last manifest referencing them was deleted with
  its job.

Mark-and-sweep from committed manifests, with a grace period longer than the
longest possible write, so a blob being written for a manifest not yet published
is never collected underneath it. This is the same hazard as Git's loose-object
pruning and the same fix.

Retention is a product decision that should be made explicitly. A tiered policy:

```text
last 5 checkpoints
hourly  × 24
daily   × 7
```

or simply *retain until the job finishes, plus a short debugging window*.

The distinction worth drawing is between two things that look alike:

| | **recovery checkpoint** | **debug / archive checkpoint** |
|---|---|---|
| goal | fast, cheap, short-lived | complete, reproducible |
| retention | minutes to hours | weeks to months |
| content | the minimum to resume | enough to reconstruct and compare |
| deleted when | the job ends | a retention policy says so |

Conflating them makes recovery expensive **and** archives incomplete. Separate
policies, separate tiers, possibly separate stores.

---

## 15. How do you know a checkpoint is good?

A checksum proves the bytes survived. It does not prove the checkpoint is
**restorable**, which is the property that matters.

The stronger gate, and one worth describing because almost nobody does:
**rebuild and compare fingerprints.**

```text
materialise the checkpoint into a fresh sandbox
      ↓
recompute the workspace fingerprint
      ↓
compare against the fingerprint recorded at checkpoint time
      ↓
mismatch → the checkpoint is not restorable, reject it now
```

Run it continuously on a sample rather than only during incidents. A checkpoint
system that has never been restored is a backup system that has never been
tested, and both fail the same way: at the worst moment, quietly.

The check has to cover **both** fingerprints — tracked source and full workspace
— because restoring the source correctly while dropping the scratch files passes
the first and fails the second, and that is precisely the defect §1 warns about.

---

## 16. What resume guarantees, and what it does not

The senior version of this question, and it is worth being precise rather than
reassuring.

Restoring state restores **the starting point**. It does not restore **the
future**, because the next model call is a fresh sample from a system that is
not deterministic.

Measured, on forked continuations from an identical reconstructed state — same
tracked source, same scratch files, same transcript, temperature 0:

```text
14 of 16 continuations reproduced the donor's next three actions exactly
 0 of 15 reached the donor's final state
```

Short-horizon behaviour is near-deterministic; long-horizon behaviour is not.
So:

> **Resume is a continuation, not a replay.** The checkpoint guarantees the
> agent restarts from the same state, not that it does the same thing.

Consequences that follow directly:

- Never describe resume as replay. The follow-up will expose it.
- Idempotency keys cannot be derived from trajectory position, because the
  resumed trajectory diverges from the original.
- If a product genuinely needs replay — audit, compliance, regression testing —
  it needs recorded **outputs**, not just recorded inputs: every tool result and
  every model response, replayed from the log instead of re-executed.

That last point is the clean answer to *"can you reproduce a run?"*: **replay
from recorded outputs, yes; reproduce by re-execution, no.**

---

## 17. Security

A checkpoint is a complete copy of a workspace that ran untrusted code, so it
inherits every concern the sandbox had, minus the sandbox.

- **Scrub credentials on write.** Short-lived scoped tokens (§design 1 §11) may
  still be live at checkpoint time; they must not be persisted. Store the
  *reference* to a credential, re-mint on resume.
- **Encrypt at rest, key per tenant.** The blob store is content-addressed, so
  identical content dedupes — which across tenants is an information leak.
  **Namespace the content address by tenant** unless you can prove the content
  is not sensitive.
- **Checkpoints are user data.** They must be covered by deletion requests, and
  content addressing makes that harder than it looks: deleting a job must not
  delete a blob another job still references.

The concrete risk if credentials are not scrubbed: a checkpoint written three
months ago carries a token that was valid three months ago. Restore it and the
token is live again, in a workspace nobody is watching. Re-mint on resume from
the secret manager — the checkpoint stores a *reference*, never a secret.

That dedup-versus-isolation tension is a good thing to raise unprompted. It is a
real tradeoff and most answers never notice it exists.

---

## 18. One mechanism, three triggers

Checkpoint and resume are usually motivated by crashes, but the same machinery
serves three purposes and the others are often more valuable day to day:

| trigger | why | difference |
|---|---|---|
| **crash recovery** | worker died | involuntary, checkpoint may be stale |
| **preemption** | scheduler needs the slot (§design 1 §8) | voluntary — take a fresh checkpoint first, so zero work is lost |
| **migration** | rebalancing, host drain, spot reclaim | voluntary, and can be planned |

Voluntary cases can checkpoint *on demand* before yielding, which makes
preemption nearly free and is what makes least-progress preemption a workable
policy at all. Building only for crash recovery leaves that on the table.

---

## 19. At 100×

| | pressure | response |
|---|---|---|
| blob store writes | 10K jobs checkpointing every N steps | content addressing already removes most; batch the manifest write |
| manifest transactions | one small atomic write per checkpoint | cheap, but it is a hot row per job — key by `job_id`, never a global counter |
| **checkpoint storm** | a cluster drain tells 10K agents to yield in 30 s | jitter the deadline, stagger the drain, rate-limit, and let jobs with a recent checkpoint skip writing a new one |
| restore stampede | a rack fails, thousands restore at once | restores read the *same* base blobs — cache them at the host, and rate-limit restores so recovery does not take the store down |
| retention | trajectories outlive jobs | tier to cold storage on a schedule, not on access |

Write-side arithmetic, to have ready:

```text
100K running agents, checkpoint every minute
  100,000 / 60          ≈ 1,667 checkpoints/s
  × 1 MB average delta  ≈ 1.67 GB/s sustained
```

That is average, not peak — which is why blobs go to object storage, metadata
stays tiny, uploads are asynchronous, and compression and dedup are not
optional. It is also why the drain case above needs explicit handling: the storm
is a multiple of this number arriving at once.

The restore stampede is the interesting one, because it is a failure that only
appears during another failure: the blob store is sized for steady-state
checkpoint writes and then asked for a burst of correlated reads. **Host-level
caching of base blobs is the fix**, and it works precisely because content
addressing made the base shared.

---

## 20. Numbers to have ready

```text
trajectory                ~40 steps, 30 min median
full archive per run      ~28 MB uncompressed for ~50 steps
tracked diff              hundreds of bytes to a few KB
transcript at the end     20–60K tokens  ≈ 100–250 KB

workspace unchanged on    ~75% of steps   → dedup does most of the work
transcript as snapshots   ~4 MB/job       → as an append-only log, ~200 KB

checkpoint interval       every 10 steps + before side effects
work lost on crash        ≤ one interval
restore                   base blob + ≤ N deltas, N bounded by re-basing
```

---

## 21. Rehearsal — answer each in 60 seconds

1. What is a checkpoint, if not a workspace snapshot?
2. Why is a half-written checkpoint worse than no checkpoint?
3. Walk through the manifest publish. What is atomic, and what is not?
4. Why content-address the blobs? What does it actually save here?
5. Why store the transcript differently from the workspace?
6. A background process was running when the worker died. What happens?
7. When do you take a checkpoint, and why only at those points?
8. How do you know a checkpoint can actually be restored?
9. Is resume the same as replay? Defend the answer with what it implies.
10. How would you support genuine replay for an audit requirement?
11. Content addressing dedupes across tenants. Is that a problem?
12. A rack fails and 2,000 jobs restore at once. What breaks?
13. Preemption versus crash recovery — why is one nearly free?
14. Why can "latest checkpoint" not mean "newest timestamp"?
15. Could a VM snapshot replace the whole design? Argue both tiers.
16. A worker crashes mid-upload, after two of three blobs. What does the next
    worker restore, and what happens to the two blobs?
