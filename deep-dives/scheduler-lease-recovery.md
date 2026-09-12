# Deep dive — Scheduler, Lease, and Failure Recovery

> Belongs to [System Design #1](../designs/agent-execution-platform.md), §8 §9 §18.

The interviewer points at the Scheduler box and asks *why*, *what if it fails*,
and *what happens at 100×*. This is the material for the next ten minutes.

**What is actually being tested.** Not whether you know the word "lease". Whether
you understand that **failure detection is unreliable**, and that every design
downstream of it is shaped by that one fact.

---

## 1. Why a lease, and not the alternatives

Three ways to give a worker a job. Two of them are wrong here, and being able to
say *why* is the answer.

**Push and forget.** Scheduler assigns, worker starts. If the worker dies the
job is lost, because nothing is watching. Fine for work that can be rerun from
the client; useless for a job carrying two hours of state.

**Push with acknowledgement.** Scheduler assigns, worker acks on completion. The
scheduler now knows about success, but still cannot distinguish *slow* from
*dead*, which is the only question that matters. Adding a timeout to this turns
it into a lease with extra steps.

**Lease: time-bounded ownership.** The worker holds the job until `now + T` and
must renew. The scheduler does not detect failure at all — it observes the
**absence of renewal**, which is a fact it can act on locally, without agreement
from anyone.

```text
scheduler                         worker
    │   assign(job, lease=30s) ──────▶ │
    │                                  │  start sandbox, run loop
    │  ◀──── renew(job) every 10s ──── │
    │                                  │
    │         (no renewal)             ✗  crash / partition / GC pause
    │  lease expires at T+30           │
    │  job becomes schedulable         │
```

The key property: **the scheduler needs no cooperation from a failed worker.**
Any design that requires the dead party to participate in its own funeral does
not work.

### Choosing the lease duration

| | short lease (10 s) | long lease (5 min) |
|---|---|---|
| detection | fast | slow — job idle for minutes |
| false expiry | frequent under GC pause or network blip | rare |
| renewal load | high | low |

**Detection time is bounded below by how long a healthy worker can be silent.**
A JVM-style GC pause or a container throttled on CPU can stall a process for
seconds. Setting the lease shorter than the worst tolerable pause converts
stalls into evictions, and evictions of a two-hour job are far more expensive
than a minute of delayed detection.

Reasonable: **lease 30 s, renew every 10 s**, so two consecutive renewals can be
lost before expiry. Recovery time is then `lease + schedule + sandbox start`, or
roughly 30 + 1 + 2 = **~33 s**, plus whatever work was done since the last
checkpoint.

---

## 2. The hard part: lease expiry does not mean the worker is dead

This is the follow-up that separates a memorised answer from an understood one.

A lease expires when renewals stop. Renewals stop when the worker is dead —
**or** when it is alive and partitioned, or paused, or its disk is slow. In
those cases the scheduler reassigns the job while the original worker is still
running it. Two executions, both believing they own the job, both writing
checkpoints and events.

You cannot fix this by making the lease longer. You can only make the damage
impossible.

### Fencing tokens

Every lease carries a **monotonically increasing epoch**. Every write that
matters carries the epoch. Storage rejects writes from a stale one.

```text
worker A   lease epoch 7   ──▶ write checkpoint (epoch 7)   ✓ accepted
           (partitioned; lease expires)
worker B   lease epoch 8   ──▶ write checkpoint (epoch 8)   ✓ accepted
worker A   (still alive)   ──▶ write checkpoint (epoch 7)   ✗ rejected, stale
```

The store does the enforcement, not the worker, because a partitioned worker
cannot be trusted to know it has been fenced. This applies to the checkpoint
store, to the job status transition, and to the event stream — anywhere a zombie
could otherwise corrupt state.

### Self-fencing, and the sandbox

Fencing protects the control plane's state. It does **not** stop a zombie from
burning money: the old sandbox is still running, still calling the model, still
executing tools with real side effects.

Two mechanisms, and you need both:

1. **The worker watches its own lease.** If renewal fails for longer than the
   lease, it kills its sandboxes without waiting to be told. This handles the
   common case, a worker that is healthy but briefly partitioned.
2. **The sandbox has its own TTL, renewed only by its worker.** If the worker
   process itself dies, nothing renews, and the sandbox self-terminates. This
   handles the case the first mechanism cannot: the worker is gone but the host
   and the VM are fine.

Belt and braces is correct here, because the two mechanisms fail in different
ways — the first depends on the worker being alive to act, the second on the
host being alive to enforce.

---

## 3. At-least-once, and what to do about duplicate effects

Given the above, the honest guarantee is **at-least-once**. Say so directly, and
say why you are not attempting exactly-once:

> Exactly-once across a process boundary and a side-effecting sandbox is not
> achievable; what is achievable is at-least-once delivery with idempotent
> effects, and I would rather put the effort into idempotency at the tool
> boundary than into a distributed protocol that still cannot cover a `git push`.

### Classify the effects

| class | example | handling |
|---|---|---|
| pure | read a file, run a test | re-run freely |
| idempotent with a key | write an object, upsert a row | dedup key derived from `(job_id, step, tool_call_id)` |
| genuinely at-most-once | `git push`, send an email, charge a card | write-ahead intent, then a dedup check at the boundary |

For the third class: record the intent in durable storage **before** performing
the effect, keyed by the deterministic tuple above. On resume, the platform
checks whether that key already completed. This does not make the effect
transactional; it makes a repeat detectable, which is what is actually needed.

### And the part specific to this workload

A **re-run model call is not a repeat — it is a new sample.** Even at
temperature 0 a serving stack does not reproduce itself: repeated identical
requests differ in content, in tool calls, sometimes in how many commands the
agent is handed. So a duplicated execution does not redo the same work slightly
wastefully; it **branches the trajectory**.

Consequences:

- Resume from a checkpoint is a *continuation*, not a replay. Never describe it
  as replay in the interview; it will not survive the follow-up.
- The trajectory must record which steps were retried, or users cannot account
  for their own runs.
- Deduplication keys must be derived from `(job_id, step)` — position in the
  trajectory — and **not** from a hash of the model's output, which is not
  stable across attempts.

---

## 4. Scheduler durability and leader election

Two questions hide here: *is the scheduler stateful*, and *what happens when it
dies*.

**Make the scheduler stateless over a durable job store.** All authoritative
state — job status, lease holder, lease epoch, expiry — lives in the store. The
scheduler is a loop that reads schedulable jobs and tries to claim them for
workers. If it dies, another instance takes over with no handoff, because there
was no in-memory state worth handing off.

### Do you need leader election?

Only if two scheduler instances can assign the same job. **Use a conditional
write instead:**

```sql
UPDATE jobs
   SET worker_id = :w, lease_epoch = lease_epoch + 1,
       lease_expires_at = now() + interval '30 seconds',
       status = 'running'
 WHERE job_id = :j
   AND (status = 'queued'
        OR (status = 'running' AND lease_expires_at < now()))
```

The row is the lock. Two schedulers race; one update applies, the other affects
zero rows and moves on. **This removes leader election entirely** — worth saying
out loud, because reaching for leader election when a conditional update will do
is a common over-design.

When you *would* want a single leader: global decisions that cannot be made per
row — bin-packing across the whole fleet, or preemption choices that need a
consistent view. Then run one leader for *placement* while keeping claiming
optimistic, so leader failure degrades packing quality rather than stopping the
platform.

### What must be durable

| component | durable? | on crash |
|---|---|---|
| Agent API | no | stateless replicas behind a load balancer |
| Scheduler | no | another instance continues from the store |
| Job store | **yes** | the platform stops; replicate, this is the one |
| Lease / heartbeat store | yes, but cheap | see §7 — different store at scale |
| Checkpoint store | **yes** | jobs cannot resume; running jobs continue, admission pauses |
| Event bus | yes | buffer at the worker, backfill; do not block the agent loop |

The line to hold: **the agent loop must never block on the observability path.**
If the event bus is down, the worker buffers and keeps going. Losing telemetry is
bad; stalling ten thousand jobs because a Kafka broker is unhappy is worse.

---

## 5. Why the job store is the queue

The natural question: *why not Kafka or SQS?*

| | fit | problem |
|---|---|---|
| Kafka | high throughput, ordered | no per-message ack with arbitrary timeout; consumer owns a partition, not a message; rebalancing a 3-hour job is not meaningful |
| SQS | visibility timeout **is** a lease | max 12 h, no priority, no per-tenant fairness, no query "what is this tenant running" |
| Job table with lease columns | queries, fairness, priority, and lease in one place | you write the polling loop; throughput is bounded by the database |

For 10K concurrent jobs the table wins easily, and the reason generalises:
**when the unit of work lives for hours, the queue is a small, slow-changing,
heavily-queried collection — which is a database, not a log.** A log is the right
answer when the unit of work is a message; here it is a machine.

Poll with a bounded, indexed query and jittered intervals so schedulers do not
synchronise:

```sql
SELECT job_id FROM jobs
 WHERE status = 'queued'
    OR (status = 'running' AND lease_expires_at < now())
 ORDER BY priority DESC, submitted_at
 LIMIT 100
```

---

## 6. Admission, fairness, and preemption

### Admission

Little's law gives the shape:

```text
concurrency = arrival rate × duration
100K jobs/day × 0.5 h avg  ≈  2,080 average concurrent
peak design point          =  10,000
```

But **duration cannot be predicted** — measured on one fixed task, runs ranged
from 3 minutes to over 6 hours. So admission cannot reserve by expected duration.
Admit against **currently free capacity** with a headroom reservation, and let
the queue absorb the rest.

### Fairness

Weighted fair queuing over **concurrent slots**, not over submissions. A tenant
that submitted once four hours ago is still consuming; fairness measured on
arrivals would let a tenant with long jobs quietly take the fleet.

```text
enterprise : 500 concurrent slots
free       : 5 concurrent slots
```

Per-tenant caps as well as weights, because weights alone do not bound the
damage when only one tenant is active and then a second arrives.

### Preemption

Only ever **to a checkpoint**, and choose the victim by **least progress**.

Preempting by priority alone destroys the most sunk cost: a job three hours in
has three hours of tokens and compute behind it, and evicting it to admit a
fresh one is the worst available trade. Least-progress preemption is also
naturally fair — it is the job that has cost the least to give up.

### Cross-layer backpressure

The subtle failure: the model service degrades, every agent slows down waiting on
it, sandbox holding time doubles, and effective capacity halves — while CPU and
memory utilisation look fine, so nothing alarms.

So **admission must read model-service latency**, not just local resources. When
model queue latency rises, lower the admission rate. Shed load by queueing new
jobs; let running ones finish, because they hold the expensive state.

---

## 7. What breaks at 100×

The best question in this section, and the answer is not "more workers".

### Heartbeat write amplification

```text
10K concurrent jobs, renew every 10 s        →  1,000 writes/s
100× that                                    →  100,000 writes/s
```

Those writes are pure overhead — they carry no information except *still alive*
— and they land on the same table serving scheduling queries. **The job store
dies of heartbeats long before it dies of jobs.**

Three fixes, in order of how much they buy:

1. **Lease per worker, not per job.** A worker holding 50 jobs renews once. The
   write rate drops by the jobs-per-worker factor immediately, and the semantics
   are unchanged: if the worker dies, all its jobs expire together, which is
   exactly what happens anyway.
2. **Separate the heartbeat store.** Leases to a KV store with native TTL;
   the job table only on state transitions. Different access pattern, different
   durability requirement, different store.
3. **Shard the job table by tenant.** Scheduling decisions are already
   per-tenant under fair queuing, so the shard key is natural, and cross-shard
   coordination is only needed for global packing.

Do (1) first. It is a schema change, not an architecture change, and it is worth
more than either of the others.

### The other two

**Event bus.** Write-heavy, centrally ordered, and the one component where load
scales with agent *activity* rather than agent *count*. Partition by `job_id` so
per-job order is preserved and nothing needs global ordering.

**Sandbox capacity** is the answer people reach for and it is the least
interesting: it scales by adding hosts, which is the easy axis. Saying so — and
naming the two above instead — is the differentiator.

---

## 8. Numbers to have ready

```text
lease duration            30 s
renewal interval          10 s          two losses tolerated
detection                 ≤ 30 s
reschedule + sandbox      ~3 s
work lost                 ≤ one checkpoint interval
end-to-end recovery       ~35 s + checkpoint interval

heartbeat writes          1K/s at 10K jobs, per-job lease
                          20/s at 10K jobs, per-worker lease (50 jobs/worker)
```

---

## 9. Rehearsal — answer each in 60 seconds

1. Why a lease rather than assignment with acknowledgement?
2. Your lease expired but the worker is alive and still running. What happens?
3. What exactly does a fencing token protect, and what does it not protect?
4. Do you need leader election for the scheduler? Argue both sides.
5. Why is the job table a better queue than Kafka here, and when does that flip?
6. A tool call has a non-idempotent side effect and the job is retried. Now what?
7. Why is resume-from-checkpoint not the same as replay?
8. The model service slows down. Trace what happens to the scheduler if nothing
   connects the two.
9. At 100× scale, what breaks first? Not sandboxes — why not?
10. How long is a job unavailable after its worker dies, and which term dominates?

If all ten come out fluently, this section is done. Move to the next deep dive
rather than polishing this one.
