# System Design #1 — Agent Execution Platform

> **Design a multi-tenant platform that can run long-running AI agents safely
> and reliably at scale.**

The concrete case is a coding agent: a user submits a task and a repository, the
agent works in a sandbox for minutes to hours, calling an LLM, running shell
commands, editing files and running tests. Thousands run at once.

**The framing that decides everything else:** this is a distributed
execution system whose workload happens to be an agent. Do not open with fifteen
minutes about LLMs.

---

## 1. Problem statement

A client submits:

```text
task
repository / environment
agent configuration
model configuration
resource requirements
```

The platform launches an isolated environment in which the agent can call an
LLM, run shell commands, read and write files, run tests, and use tools. A job
lasts from several minutes to several hours.

Must support:

```text
10K+ concurrent agent jobs
strong isolation
checkpoint / resume
retries
resource scheduling
observability
cost accounting
```

---

## 2. Clarify before designing

Ask only the questions whose answers change the architecture.

| question | what it changes |
|---|---|
| Is agent-generated code untrusted? | container vs microVM; whether kernel escape is in scope |
| How long can one job run? | minutes → stateless workers; hours → checkpointing is mandatory |
| Is resume required, or is restart acceptable? | the entire state layer exists only if resume is required |
| Network egress allowed? | egress proxy + allowlist, or air-gapped with a package mirror |
| Fixed tools or user-defined? | user-defined tools is a second untrusted-code problem |
| Do we own inference, or call a model service? | if we own it, GPU scheduling joins the design |
| Real-time streaming of trajectory? | polling vs an event bus |
| Deterministic replay required? | forces recording inputs, seeds and tool outputs, not just a log |

**State the scope out loud:**

> I'll assume a coding-agent platform executing potentially untrusted code, jobs
> up to several hours, resume required, and LLM inference provided by a separate
> model-serving layer.

That last clause is what stops this from becoming two designs at once, and it is
the honest split in real systems.

---

## 3. Non-functional requirements

### Isolation

One agent must not reach another's:

```text
filesystem   memory   credentials   network namespace   processes
```

The threat model is deliberate escape, not accidental interference: the code
being run was written by a model following instructions a stranger supplied.

### Reliability

A worker crash loses at most one checkpoint interval, not the job.

```text
99.9% platform availability
at-least-once job execution
checkpoint recovery
```

At-least-once is the right choice here and it is **not free** when the workload
is stochastic — see §18.

### Scalability

```text
100K jobs/day
10K concurrent at peak
median duration 30 min
p99 duration 3 h
```

### Startup latency

```text
sandbox start   p50 < 2 s     p99 < 10 s
```

An agent that takes 30 seconds to get its first shell prompt feels broken even
when the job runs for an hour.

### Observability

```text
job state       tool calls      LLM calls       resource usage
sandbox events  cost            failure reason
```

The failure reason must distinguish *the agent gave up* from *we dropped it*.

---

## 4. Data model

Four entities. The boundaries between them matter more than the fields.

### Job — what the user asked for

```text
Job {
  job_id, tenant_id
  task_spec          prompt, repository reference, entry conditions
  agent_config       loop policy, step limit, tool allowlist
  model_config       model id, revision, sampling parameters
  resource_request   cpu, memory, disk, timeout
  status             queued | running | succeeded | failed | cancelled
  latest_checkpoint
}
```

### Execution — one attempt at it

```text
Execution {
  execution_id, job_id, attempt
  worker_id, sandbox_id
  start_time, end_time
  status, termination_reason
  resumed_from_checkpoint
}
```

A job has many executions. Keeping them separate is what makes retries
auditable: *this job succeeded* and *this job succeeded on the third attempt
after two worker evictions* are different facts, and only the second explains
the bill.

### Agent state — what a checkpoint has to contain

```text
AgentState {
  message_history        the full conversation, verbatim
  workspace_snapshot     tracked source diff + files the agent itself created
  tool_state             open handles, background processes, env mutations
  environment_metadata   base image digest, not a live container id
  step
}
```

**A checkpoint is not a message history.** §12.

### Event — the append-only trajectory

```text
Event {
  event_id, job_id, execution_id, step, seq
  kind        model_call | tool_call | state_change | limit | error
  payload_ref inline if small, object store if not
  timestamp
}
```

Append-only and ordered per execution. This is both the client's live stream and
the permanent record; one source for both is what keeps them from disagreeing.

---

## 5. High-level architecture

```text
                   ┌──────────────────┐
Client ───────────▶│ Agent API        │◀──── status, event stream
                   └────────┬─────────┘
                            │
                            ▼
                   ┌──────────────────┐
                   │ Job Store        │
                   └────────┬─────────┘
                            │
                            ▼
                   ┌──────────────────┐
                   │ Scheduler        │
                   └────────┬─────────┘
                            │
                 ┌──────────┴───────────┐
                 ▼                      ▼
          ┌────────────┐         ┌────────────┐
          │ Worker     │   ...   │ Worker     │
          │            │         │            │
          │ Sandbox    │         │ Sandbox    │
          │ Manager    │         │ Manager    │
          └─────┬──────┘         └─────┬──────┘
                │                      │
                ▼                      ▼
          ┌────────────┐         ┌────────────┐
          │ Agent      │         │ Agent      │
          │ Runtime    │         │ Runtime    │
          └─────┬──────┘         └─────┬──────┘
                │
          ┌─────┴──────────────┐
          ▼                    ▼
    Model Service         Tool / Shell
    GPU inference         execution
```

Two cross-cutting services:

```text
Checkpoint / Object Store
Telemetry / Event Pipeline
```

---

## 6. Control plane vs data plane

The split that most of the failure story depends on.

| | control plane | data plane |
|---|---|---|
| owns | job API, metadata, scheduling, quota, admission, retries, lifecycle | agent runtime, sandbox, tool execution, filesystem, model calls |
| runs user code | **never** | always |
| scaling | small, stateful, consistent | large, stateless, disposable |
| failure | outage of the platform | loss of one job at most |

**Why separate:** a broken or malicious agent must not be able to affect the
scheduler. If the same process both runs untrusted code and decides what runs
next, one escape is the whole platform.

The practical consequence: the control plane never parses agent output, never
executes a tool, and never loads a workspace. It moves opaque blob references.

---

## 7. Job lifecycle

```text
POST /jobs
    ↓
PENDING
    ↓
admission control            quota, budget, global queue limit
    ↓
QUEUED
    ↓
scheduler chooses worker
    ↓
sandbox created
    ↓
RUNNING
    ↓
agent loop
    │
    ├── LLM call
    ├── tool execution
    ├── observation
    └── checkpoint
    ↓
SUCCEEDED / FAILED
```

Crash path:

```text
worker crash
    ↓
lease expires
    ↓
scheduler detects orphaned job
    ↓
load latest checkpoint
    ↓
restart on another worker
```

Nothing in this path is synchronous from the client's point of view. Long jobs
make every API call a submission or a query, never a wait.

---

## 8. Scheduling

The scheduler weighs:

```text
CPU   RAM   sandbox slots   model locality
tenant quota   job priority   expected runtime
```

But coding agents have a property that changes the answer:

> **Most of the time, an agent is not using a GPU.**

The loop is:

```text
LLM inference  →  CPU tool execution  →  wait for tests  →  LLM inference
```

So:

**Do not bind a GPU to each agent.**

```text
10K agent workers
       │
       │ shared requests
       ▼
GPU inference pool
```

Agent runtime and inference serving are decoupled and scale independently. They
have completely different bottlenecks — agents are memory-bound and idle;
inference is bandwidth-bound and saturated.

Two further rules that come from how these jobs actually behave:

- **Bin-pack on memory**, not CPU. Memory binds first (§17) and agents idle
  while waiting on the model, so CPU can be oversubscribed 3–4×.
- **Preempt the young, not the old.** A job three hours in has three hours of
  cost in it. Preempt by least progress, and only to a checkpoint.

And one honest limitation to state: **duration cannot be estimated in advance.**
Measured on a single fixed task, run lengths ranged from 3 minutes to over 6
hours. Any admission or packing decision that depends on a duration estimate
will be wrong often enough to matter.

---

## 9. Queue and lease model

> Deep dive: [Scheduler, lease, failure recovery](../deep-dives/scheduler-lease-recovery.md)

Choose: **durable queue + worker lease.**

```text
job J → worker W
lease expiration = now + 30 s
```

The worker heartbeats to renew. If heartbeats stop, the lease expires and the
job becomes schedulable again. That is the entire crash-recovery mechanism, and
it is simpler than anything involving worker-side durability.

### At-least-once execution

A job may briefly be executed twice — the classic case is a worker that is
alive but partitioned, still working, while its lease expires. So the runtime
must be built for **idempotency**, not for exactly-once.

Say it directly:

> I'd prefer at-least-once execution with idempotent lifecycle operations rather
> than building exactly-once distributed execution semantics.

Then the part that is easy to miss, because it is specific to this workload:
at-least-once interacts badly with a stochastic runtime. §18.

---

## 10. Sandbox design

Three candidates, and the answer is not one of them universally.

| | boundary | startup | overhead | weakness |
|---|---|---|---|---|
| Docker / runc | namespaces, cgroups, seccomp | ~100 ms | lowest | shared kernel — one bug is a cross-tenant escape |
| gVisor | userspace kernel | ~200 ms | 10–30% syscall | some syscalls unimplemented; compatibility surprises |
| Firecracker microVM | hardware virtualisation | ~150 ms + guest boot | ~5% memory | heavier image and device plumbing |

Design:

```text
trusted internal workloads   →  container / gVisor
untrusted arbitrary code     →  Firecracker-like microVM
```

**Do not claim one option is universally right.** Naming the tradeoff is the
point; a side picked without one is not.

### Meeting the startup budget

A pool of **pre-booted microVMs** with the base image already mapped. Allocation
becomes attach-the-workspace rather than boot-a-machine, which is what keeps p50
under 2 s.

Size the pool to the **peak arrival rate**, not peak concurrency. Sizing it to
concurrency is a common and expensive mistake — it buys a fleet of idle VMs.

---

## 11. Network isolation

Default policy:

```text
default deny
```

Egress through a per-tenant proxy:

```text
Sandbox  →  network policy / egress proxy  →  approved domains
```

Provide a package mirror, so that "no internet" does not mean "cannot install a
dependency" — otherwise every user works around the control and you have neither
isolation nor convenience.

Credentials:

```text
short-lived scoped tokens
```

Never a long-lived API key on the sandbox filesystem. Anything written there
should be assumed exfiltrated.

---

## 12. Checkpoint and resume

The wrong design:

```text
checkpoint = git diff
```

Insufficient, because the agent's state also includes:

```text
message history
untracked files
tool outputs
environment state
```

So:

```text
Checkpoint
├── agent context         message log, verbatim
├── tracked repo state    canonical diff against the base commit
├── untracked workspace   scratch files the agent itself created
├── tool / runtime state  open handles, background processes, env
└── metadata              base image digest, step, model revision
```

**Why untracked files are not optional.** Restore the transcript and the tracked
source but drop the scratch files, and you get an agent whose history
contradicts what it can see: it remembers writing `repro.py`, reads it back, and
is told the file does not exist. What follows is not a resumed job, it is a
confused one. This is the failure mode that makes checkpointing look like it
works in testing and fall apart on real trajectories.

Storage:

```text
metadata          → database
large snapshots   → object storage, content-addressed
```

Workspace deltas dedupe well across attempts of the same job.

Snapshotting every step is too expensive. Checkpoint:

```text
incrementally
every N steps
on important state transitions
before risky actions
```

**Take checkpoints at step boundaries only.** That is where the agent's state is
coherent — the last message is an observation, not a half-finished action. A
tool call interrupted by a crash is re-run from the previous boundary, which is
exactly why tool idempotency matters.

---

## 13. Event log

The runtime emits append-only events:

```text
JobStarted        SandboxCreated
ModelRequested    ModelCompleted
ToolStarted       ToolCompleted
CheckpointCreated
JobSucceeded / JobFailed
```

Into:

```text
Kafka / Pulsar  →  stream processors  →  metrics / traces / analytics
```

Why it matters: debugging, billing, replay and post-hoc analysis all read from
this one log. Building it later means rebuilding the runtime.

One design rule worth stating: **instrumentation must not enter the agent's own
context.** If the probe that records repository state writes into the agent's
observation, the measurement has changed the thing being measured. Record out of
band, in the worker, never in the loop.

---

## 14. Observability

Do not answer "logs, metrics, traces". Name the agent-specific telemetry.

**Per job**

```text
step count        tokens in / out       LLM latency
tool latency      sandbox startup       context size
retry count       checkpoint size       failure category
cost
```

**Per platform**

```text
queue depth              scheduler latency
worker utilization       sandbox capacity
GPU inference utilization
job success rate
```

Two that are easy to forget and diagnostic out of proportion to their cost:
**context size over time** (it grows monotonically and predicts both cost and
context-limit failures) and **retry count per step** (§18).

---

## 15. Backpressure

Suppose 100K jobs arrive at once. You cannot create sandboxes without limit.

Admission controller:

```text
tenant quota
global queue limit
resource reservation
```

And when the inference service degrades:

```text
model queue latency ↑
        ↓
agent scheduler lowers admission rate
```

This is **cross-layer backpressure**, and it is the thing to name. The failure
without it: the platform keeps admitting jobs, every running agent slows down
waiting on the model, sandbox holding time doubles, and capacity collapses while
utilisation looks fine.

Shed load by queueing *new* jobs. Let running ones finish — they hold the
expensive state.

---

## 16. Multi-tenancy

Per tenant:

```text
max concurrent jobs
token budget
CPU-hour budget
priority
model access
```

For example:

```text
enterprise tenant : 500 concurrent
free tenant       : 5 concurrent
```

The scheduler applies **weighted fair queuing** so that one tenant submitting
5,000 jobs cannot starve everyone else. Fairness is on concurrent jobs, not on
submissions — a long job consumes capacity for hours after it was admitted.

---

## 17. Capacity estimation

Do this arithmetic out loud. It decides the shape of everything else.

### Sandboxes

```text
10,000 concurrent × 2 vCPU, 4 GB   →  20,000 vCPU, 40 TB RAM
at 64 vCPU / 256 GB per host       →  ~310 hosts
```

**RAM binds before CPU.** But agents spend most of their time waiting on the
model or on tests, so with average CPU utilisation around 20%:

```text
effective CPU demand ≈ 4,000 cores
```

That is the opening for the real discussion: **oversubscription against
latency**. Oversubscribe and a burst of simultaneous test runs queues on CPU;
do not, and you buy five times the fleet. The honest answer is oversubscribe on
CPU, never on memory, and keep a headroom pool for bursts.

### Model traffic

Roughly 40 steps per job over 30 minutes → one model call per agent per ~45 s.

```text
10,000 agents / 45 s          ≈ 220 requests/s
prompt grows each step, reaching 20–60K tokens late in a job
output 300–5,000 tokens per step

naive prefill   220 × 30K     ≈ 6.6M tokens/s      ← not buildable
with prefix caching, only the delta is new:
                220 × 2K      ≈ 440K tokens/s
decode          220 × 800     ≈ 176K tokens/s
```

**Prefix caching is load-bearing, not an optimisation.** The agent resends its
whole transcript every step, so without it prefill is quadratic in trajectory
length and the cluster is fifteen times larger. If one number in this design is
worth remembering, it is this one.

It follows that the model gateway must **route for cache affinity** — send a
job's successive calls to the replica that already holds its prefix. Random
routing throws the cache away and restores the 6.6M figure.

### Trajectory storage

```text
~40 steps × ~25 KB command and output, compressed  ≈ 1 MB/job
100K jobs/day  →  100 GB/day  →  ~36 TB/year
```

Hot for a week, then cold object storage.

---

## 18. Failure modes

| failure | handling |
|---|---|
| API server crash | stateless replicas |
| Scheduler crash | durable job state + leader election |
| Worker crash | lease expiry + checkpoint resume |
| Sandbox crash | recreate sandbox from checkpoint |
| Sandbox OOM | classify as resource failure, surface the limit, do not blindly retry |
| Model timeout | retry with a bounded policy |
| Tool hangs | per-tool timeout |
| Agent loops forever | step / time / token budget |
| Object store unavailable | pause checkpoint-dependent operations |
| GPU inference overloaded | backpressure (§15) |
| Poison job | cap attempts, quarantine, keep the trajectory |

### The one worth dwelling on: a retried model call changes the trajectory

Standard at-least-once says: on doubt, run it again. For an agent platform that
is subtly wrong.

A model call that was generated but whose response was lost is **not idempotent
to re-send**. Even at temperature 0, a serving stack does not reproduce itself:
repeated identical requests differ in content, in tool calls, and sometimes in
how many commands the agent is handed. A retried step therefore samples the
model a second time and keeps the later draw. The job is not resumed — it is
branched.

So the telemetry must record, per step:

```text
request hash
attempt number
retry reason
```

And the policy should prefer **short client timeouts with a bounded attempt
count** over long ones. A dead connection detected in 90 seconds and retried
once is far better than one detected in ten minutes, and better than an
unbounded wait.

### A transport failure that looks exactly like model instability

Worth knowing because it is common and misdiagnosed: agent steps routinely
generate thousands of tokens, which is minutes of wall clock. A **non-streaming**
request sends no bytes until its last token, and any proxy with a response
timeout — a frequent default is around 100 seconds — kills it and returns an
error page, while the model finishes the request normally and records success.

The symptom is intermittent hangs that correlate with nothing, and the diagnosis
is not in the model's metrics because the model was fine. Mitigations: stream
every call, colocate the agent runtime with inference rather than crossing a
WAN, and bound the read with an explicit idle timeout rather than a total one.

### What breaks first at 10×

The job table and the event bus — both write-heavy and centrally ordered — long
before sandbox capacity does. Sandboxes are the obvious bottleneck and the wrong
answer.

---

## 19. Cost controls

An agent can burn tokens without limit. Per job:

```text
max_steps
max_tokens
max_wall_time
max_cost
budget_remaining
```

The runtime terminates with an explicit reason:

```text
BudgetExceeded
```

A limit hit is **data, not an error** — it should be a distinct terminal state,
not a failure, because the difference matters for both the user and the metrics.

Report cost per **job**, including failed attempts. The number that matters is
what the job cost, not what its successful attempt cost.

---

## 20. Tradeoffs

| decision | chosen | given up | choose otherwise when |
|---|---|---|---|
| microVM isolation | strong boundary | ~5% memory, pool complexity | trusted first-party code only |
| checkpoint every N steps | bounded loss | storage, pause cost | jobs are short enough to just restart |
| at-least-once | simple recovery | duplicate effects, resampling | strict effects → at-most-once + manual resume |
| stateless workers | trivial rescheduling | all state on the write path | latency-critical → sticky workers, weaker recovery |
| external inference | independent scaling | no control over batching or cache | inference is the bottleneck → own it |
| append-only events | one source of truth | storage growth | cost-sensitive → sample non-essential events |
| CPU oversubscription | 5× smaller fleet | tail latency under burst | latency SLO is tight |

---

## 21. How this connects to reliability work

If asked *how would you debug agent reliability?* — the platform already
produces what is needed, and the answer follows from the design rather than
arriving as a detour:

```text
runtime event stream
      ↓
trajectory state representation
      ↓
compare repeated executions
      ↓
detect consequential divergence
      ↓
checkpoint / replay / fork
```

The observation that motivates it: repeated executions of the same task, same
model, temperature 0, converge on an identical intermediate repository state and
then reach different final states. Variation is not failure — the open question
is which variation changes the outcome.

And the forward-looking version, which is what the checkpoint machinery makes
possible:

```text
risk rising
   ↓
runtime checkpoint
   ↓
fork alternatives
   ↓
continue the best branch
```

---

## 22. Telling it in 45 minutes

```text
0–5     requirements + scale
5–10    high-level architecture
10–20   job lifecycle + queue + scheduler
20–30   sandbox + isolation + checkpoint
30–37   failure handling + observability
37–42   scale / capacity / cost
42–45   tradeoffs + extensions
```

Never open with fifteen minutes on LLMs. This is a **distributed execution
system** whose workload happens to be an agent.

---

## 23. Questions the design has to survive

1. **Why not just Kubernetes Jobs?** Pod lifecycle assumes short, restartable
   work; no checkpoint story for a stateful hours-long workspace; the sandbox
   boundary is weaker than needed for untrusted code. Kubernetes is a reasonable
   substrate for the worker pool, not a substitute for the scheduler.
2. **How do you execute untrusted code safely?** §10 and §11: microVM, default-
   deny egress, short-lived scoped credentials, and no long-lived secrets in the
   sandbox.
3. **A worker dies halfway through a 2-hour run?** Lease expiry, reschedule from
   the last checkpoint, lose at most one checkpoint interval. §9.
4. **How do you prevent duplicate execution?** You do not prevent it; you make
   it safe. At-least-once plus idempotent lifecycle operations, with idempotency
   keys at the tool boundary for effects that must not repeat. §9, §18.
5. **How do you checkpoint efficiently?** Incremental, content-addressed,
   step-boundary only, every N steps and before risky actions. §12.
6. **How do you schedule 10K long-running agents?** Memory bin-packing, CPU
   oversubscription, lease-based assignment, weighted fair queuing, preempt by
   least progress. §8, §16.
7. **Should every agent have a GPU?** No — agents are idle most of the time.
   Decouple the runtime from a shared inference pool. §8.
8. **What if the model cluster is overloaded?** Cross-layer backpressure: lower
   admission, queue new jobs, let running ones finish. §15.
9. **What must be persisted to reproduce an execution?** Model id and revision,
   serving configuration, sampling parameters, every tool output, and which
   steps were retried. Then say the true thing: this yields a *replayable*
   trajectory, not a *reproducible* one, because the serving stack is not
   deterministic at temperature 0.
10. **How does this evolve from 100 to 100K agents?** 100: one process, one
    queue, containers. 10K: split control and data planes, lease-based
    scheduling, checkpointing, microVMs. 100K: shard the job store, partition
    the event bus, regionalise, and colocate agent workers with inference.

---

## 24. What to build first

The **worker lease** and the **checkpoint format**. Everything else can start as
a single process and a queue; those two are the pieces that are painful to
change once real jobs are running.
