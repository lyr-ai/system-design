# System Design #4 — Secure Sandbox Service for Untrusted Agent Code

> **Design a multi-tenant service that executes arbitrary code produced by AI
> agents, at large scale, with a strong isolation boundary and a startup
> latency low enough for interactive agents.**

The concrete case is the Sandbox box of [design #1](agent-execution-platform.md).
An agent receives a repository, edits files, installs packages, compiles, runs
tests, starts local services, and executes code it copied from an issue, a
README, a dependency, or a tool response. None of that code is trusted, and
some of it was written to get out.

The service has to make this ordinary:

```text
create an isolated environment
      ↓
materialise a workspace
      ↓
run arbitrary commands for minutes or hours
      ↓
allow only explicitly granted external capabilities
      ↓
return artifacts and recorded effects
      ↓
destroy the environment, and everything it was allowed to do
```

at ten thousand concurrent sandboxes, without every agent start being a
thirty-second VM boot.

**The framing that decides everything else:** the isolation technology is the
smaller half of the problem. A sandbox has four boundaries, and the mechanism
most people spend the interview on is only the first:

```text
execution boundary      can the code escape the sandbox?
resource boundary       can one sandbox exhaust the host, or another tenant?
authority boundary      what can the code reach *without* escaping?
lifecycle boundary      what survives after the sandbox is gone?
```

The threat model is *adversarial by proxy*: no attacker is present, but the
input distribution includes whatever an attacker chose to put in a GitHub
issue. Assume anything reachable from inside the sandbox will eventually be
read, modified or exfiltrated.

> **Isolation contains computation. Capabilities contain authority. Both are
> required, and the second is where most real incidents happen.**

The mechanisms — why a shared kernel is insufficient here, gVisor against
Firecracker, filesystem/network/credential isolation, snapshot restore and its
entropy caveat — are in [Sandbox and isolation](../deep-dives/sandbox-isolation.md).
This design starts one level up: how those mechanisms compose into a service
that #1 can call.

---

## 1. Problem statement

The execution platform submits:

```text
SandboxRequest {
  tenant_id, job_id
  image_ref                  by digest
  workspace_ref
  resources                  cpu, memory, disk, pids, wall_time
  capabilities               network policy, package sources, secrets,
                             effect permissions
  lifecycle                  ephemeral | checkpointable; ttl
  class                      interactive | batch | heavy_build
}
```

and receives:

```text
Sandbox { sandbox_id, generation, endpoint, state }
```

Operations:

```text
Create · Exec · Upload · Download · Checkpoint · Restore · Destroy
```

The word that carries the design is **capabilities**. A sandbox is not safe
because the code cannot escape its VM; if that VM has an unrestricted network
path and an organisation-wide token, no escape is needed.

---

## 2. Clarify before designing

| question | what it changes |
|---|---|
| Are tenants mutually untrusted? | per-tenant VM, never reuse across tenants, tenant-namespaced caches |
| Must sandboxes run containers themselves? | microVM with nested virtualisation, and rules out gVisor for that class |
| Is egress required at all? | the whole capability plane exists only if some egress is; if none, air-gap plus a mirror |
| Checkpointable, or ephemeral only? | workspace overlay design; snapshot tier; the authority-revalidation rule on restore |
| Interactive or batch? | the startup SLO, and whether warm pools are worth their idle cost |
| GPU inside the sandbox? | device passthrough, a different isolation story; assume not |
| Are irreversible effects performed from inside? | if yes, the effect gateway is in scope; if no, only network and secrets |
| What is the sandbox allowed to persist? | the lifecycle boundary: nothing but the returned workspace |

**State the scope out loud:**

> I'll assume mutually untrusted tenants, arbitrary Linux toolchains including
> nested containers for some workloads, egress needed for package installs and
> a few approved hosts, checkpointable sandboxes, an interactive startup
> budget of two seconds at p50, and no GPUs inside.

---

## 3. Non-functional requirements

### Security, stated as invariants

```text
a sandbox compromise does not imply a host compromise
tenant A cannot read tenant B's memory, filesystem, credentials, or the host's
network is deny-by-default
no ambient cloud, production, or tenant credentials inside the sandbox
resource exhaustion stays local to the sandbox
destroy means tenant state and tenant authority are no longer reachable
```

### Availability, deliberately weaker

A sandbox that cannot start is a failed job. A sandbox that starts without the
requested isolation policy is a security incident. So:

> **Fail closed on policy and isolation. Fail independently on execution.**

A policy-engine outage stops new sandboxes; it never produces sandboxes with a
default policy.

### Scale and latency

```text
100K jobs/day, 10K concurrent at peak
lifetime seconds to hours
time-to-ready    p50 < 2 s    p99 < 10 s
```

### Auditability

Every grant, every egress decision, every effect intent, and every lifecycle
transition is logged with `sandbox_id`, `tenant_id`, `policy_version`.

---

## 4. Data model

Six entities, and the boundaries between them are the security argument.

### Sandbox — an identity with a generation

```text
Sandbox {
  sandbox_id
  generation                 increments on restore; dead after destroy
  tenant_id, job_id
  host_id
  policy_version
  state
  created_at, ttl
}
```

`generation` is what stops a stale client — or a stale worker from #1 — from
sending commands to an identity that has been destroyed and its resources
reused.

### Policy — decided, not enforced, by the control plane

```text
Policy {
  policy_version
  network        allowlist, dns mode, deny_private_ranges, max_egress_bytes
  secrets        capability-scoped requests the broker may honour
  effects        which effect classes the gateway may accept
  resources      the limits the host will apply
}
```

### Grant — an issued capability

```text
Grant {
  grant_id
  sandbox_id, generation
  capability                 github.read(repo=X) · egress(host) · effect(class)
  expires_at
  revoked_at
}
```

A grant is bound to a sandbox *and* a generation, is short-lived, and is
indexed by `sandbox_id` so that revocation is one logical operation.

### Host

```text
Host {
  host_id
  capacity, allocated
  cached_images[], cached_snapshots[]
  security_status            healthy | quarantined | reimaging
  isolation_modes            microvm | gvisor
}
```

### Exec — idempotent at the API

```text
Exec { sandbox_id, generation, command_id, argv, cwd, timeout }
```

### Workspace

```text
Workspace { overlay_ref, base_image_digest, quota, returned_delta_ref }
```

The boundary that matters most: **policy is decided by the control plane,
grants are issued by the capability plane, enforcement happens on the host
and at the proxy.** No component both decides and enforces, so an escape
lands on a component that cannot change what it was allowed to do.

---

## 5. High-level architecture

```text
                    Agent Execution Platform (#1)
                              │  SandboxRequest
                              ▼
               ┌──────────────────────────────┐
               │     SANDBOX CONTROL PLANE    │   never runs tenant code
               │  API / authn · Policy Engine │   never holds tenant secrets
               │  Placement · Lifecycle       │
               └───────────────┬──────────────┘
                               │ desired sandbox + policy
           ┌───────────────────┴────────────────────┐
           ▼                                        ▼
  ┌─────────────────────┐                ┌──────────────────────┐
  │     SANDBOX HOST    │                │   CAPABILITY PLANE   │
  │  host agent         │                │  egress proxy        │
  │  image cache        │                │  credential broker   │
  │  snapshot cache     │                │  effect gateway      │
  │  ┌───────────────┐  │   explicit,    │  audit log           │
  │  │   microVM     │──┼── scoped, ────▶│                      │
  │  │  workspace    │  │   revocable    └──────────────────────┘
  │  │  agent code   │  │
  │  └───────────────┘  │
  └──────────┬──────────┘
             ▼
     Artifact / Checkpoint Store
```

Three planes, and each is deliberately ignorant of something:

- the **control plane** decides policy and never executes tenant code or holds
  tenant secrets;
- the **host** enforces limits and isolation and never decides what authority
  a sandbox gets — it applies a policy it was handed;
- the **capability plane** issues and enforces grants and never trusts the
  sandbox's claim about who it is — identity was established at creation and
  bound to the generation.

An escape from the microVM lands on a host that holds no tenant authority. An
escape from the host lands on planes that do not trust hosts. That chain is
the design.

---

## 6. Control plane vs data plane

| | control plane | capability plane | host (data plane) |
|---|---|---|---|
| decides authority | yes | no | no |
| issues authority | no | yes, from policy | no |
| enforces | no | egress, effects | limits, isolation |
| runs tenant code | never | never | inside the VM only |
| holds tenant secrets | never | brokered, short-lived | never at rest |
| trusts | callers it authenticated | grants it issued | the control plane's policy |

The host agent deserves one more sentence: it is the most exposed trusted
component, one boundary away from the code. It carries a narrowly scoped
materialisation grant — pull this image digest, attach this workspace — and a
policy token for the sandboxes it runs. It cannot read other tenants'
workspaces, mint credentials, or write control-plane state.

---

## 7. Lifecycle

```text
REQUESTED → POLICY_RESOLVED → PLACED → MATERIALISING → READY → RUNNING
                                                                  │
                                                     ┌────────────┼──────────┐
                                                  EXEC       CHECKPOINTING   │
                                                                             ▼
                                                                       TERMINATING
                                                                             │
                                                                          DESTROYED
```

### Create

Control plane: authenticate the caller; resolve the policy; validate image
digest and limits; choose a host (§11); allocate `sandbox_id` and generation 1.

Host: restore a boot snapshot or boot clean; **reseed entropy, resync the
clock, inject fresh identity**; attach a fresh writable overlay; apply CPU,
memory, PID, disk, I/O and network limits; establish the capability-plane
identity; report `READY`. Nothing executes before `READY`.

### Exec

Not a new sandbox. `command_id` makes submission idempotent; `generation`
makes a stale submission fail rather than land somewhere else.

### Destroy — and the order is the point

```text
revoke all grants for (sandbox_id, generation)     ← first
      ↓
stop the microVM
      ↓
detach and discard the writable overlay
      ↓
release resources; mark the generation dead
```

Revocation comes first. Destroying compute while a credential stays valid for
five more minutes means the lifecycle did not end; it moved to wherever the
credential was copied.

---

## 8. Deep dive — the blast-radius chain

The default is a Firecracker-class microVM per sandbox; the deep dive makes
the case. The service-level consequence is the chain of boundaries a
compromise has to cross, and the design assumes each one may fail:

```text
sandbox compromise
      ↓  microVM boundary
host agent
      ↓  host holds no tenant authority
host
      ↓  planes do not trust hosts
control / capability planes
```

### The host carries no tenant authority

Not a design nicety — the thing that makes a VM escape survivable. A host must
not be able to read arbitrary tenant repositories, reach production services,
mint credentials, or write control-plane state. If it can, the microVM is the
only boundary, and one kernel bug is the whole platform.

### Quarantine, without deliberation

Any signal consistent with an escape:

```text
unexpected hypervisor behaviour       host integrity check failure
policy daemon modified                forbidden device access
cross-sandbox memory anomaly          host-level egress not attributable to a sandbox
```

→

```text
host → QUARANTINED
stop placement to it
revoke every grant for every sandbox on it
capture forensic state where safe
terminate or migrate its sandboxes, per tenant policy
reimage before it returns to the fleet
```

Do not determine whether it was a false positive while continuing to place
new untrusted tenants there. Quarantine is cheap; the alternative is not.

### Tenant anti-affinity

Mutually untrusted tenants are not co-located on a host beyond a configured
density, so that a single host compromise bounds the number of tenants
exposed. This is a placement constraint that overrides packing efficiency.

---

## 9. Deep dive — no ambient authority

The kernel boundary answers *can the code escape*. The capability plane
answers *what can the code do without escaping* — and that question decides
the real blast radius more often.

A sandbox starts with **nothing**:

```text
not:   sandbox starts + env holds credentials + network reaches internal services
but:   sandbox starts + no network + no credentials + an identity it can present
```

Every external capability is then **explicit, scoped, short-lived, auditable,
revocable**, and bound to `(sandbox_id, generation)`.

> **No ambient authority.** Access exists because this sandbox was granted
> this capability, not because it runs on the right network or host.

### Network

Default deny, then make the denial survivable — a control everyone works
around is not a control:

```text
NetworkPolicy {
  dns                  controlled resolver          or DNS is the exfiltration channel
  allow                package mirror, approved registries, explicit hosts
  deny_private_ranges  true                         including cloud metadata
  max_egress_bytes     per sandbox
}
```

Traffic leaves through an authenticated egress proxy that logs `sandbox_id`,
`tenant_id`, destination, bytes, decision, `policy_version`. **Destination
allowlisting is not enough**: 400 MB posted to an allowed host is still
exfiltration, so volume and pattern are signals, not just destination.

### Secrets

A request is for a capability, not a credential:

```text
sandbox S requests   github.read(repo=X)
not                  the user's GitHub token
```

The broker either performs the operation on the sandbox's behalf or mints a
credential that can do only that operation, for minutes, bound to `S`. Nothing
long-lived is ever inside the VM. Prefer a metadata endpoint the sandbox
queries over a value injected into the environment: an endpoint can be
revoked the moment the job ends; an environment variable cannot be taken back.

### Irreversible effects

```text
merge PR · send email · modify production · delete a cloud resource · publish a package
```

go through the guarded effect gateway from #1: the sandbox submits an
*intent*, the gateway checks policy, performs the action, records it. The
sandbox never holds the credential for any of these.

> The strongest credential is the one that never entered the sandbox.

### Revocation is O(1)

Grants are indexed by `sandbox_id`. `Destroy(S)` implies `RevokeAll(S)` as
one logical operation, however many grants were issued in six hours, and the
proxy and broker consult the revocation before every use.

---

## 10. Deep dive — resource isolation is also security

Without enforcement, a sandbox attacks the platform without escaping anything.

| abuse | control |
|---|---|
| fork bomb | PID limit per sandbox |
| memory bomb | hard memory limit; host headroom so simultaneous OOMs do not push the host into reclaim collapse |
| disk bomb | overlay quota; package caches shared read-only or mediated — otherwise the host's cache becomes tenant-controlled persistent state |
| infinite execution | command timeout < sandbox TTL; the lifecycle manager owns the TTL, so code inside cannot extend its own lease |
| network flood | egress bandwidth cap plus the byte budget |
| I/O saturation | per-sandbox I/O bandwidth |

### Noisy neighbours are a class problem

Quotas stop monopolisation; they do not stop a heavy build from degrading an
interactive agent's latency through host scheduling and I/O contention. At
scale, declare workload classes —

```text
interactive · batch evaluation · heavy build
```

— and do not pack incompatible classes on one host because the numbers fit.
Interactive sandboxes get reserved hosts; batch is preemptible to a
checkpoint, where its semantics allow (#3 says when they do not).

---

## 11. Deep dive — two-second starts without weakening the boundary

The mechanisms are in the deep dive: image locality, snapshot restore, warm
pools, and the fact that the dominant startup term — pulling the image — has
nothing to do with isolation. At the service level:

```text
placement  →  image present? (else lazy pull)  →  snapshot present? (restore, else boot)
           →  reseed + identity  →  attach workspace  →  attach policy  →  READY
```

### Placement optimises time-to-ready

```text
hard     resources fit · isolation mode · architecture · tenant anti-affinity
         · host not quarantined · workload class compatible
soft     image cached · snapshot cached · workspace locality · fragmentation
```

A host with the image and snapshot resident starts a sandbox in hundreds of
milliseconds. An empty host that must pull 8 GB takes minutes. So a host with a
quarter of the free capacity and the image wins — the scheduler is scoring
**time-to-ready**, not free CPU.

### Warm pool: size to arrivals, never to concurrency

```text
100K jobs/day  ≈ 1.2 starts/s average, ~10/s at peak
replenish a warm VM in ~5 s
pool of ~50–100 absorbs the peak
```

Sizing to 10K concurrent buys ten thousand idle VMs to serve jobs that
already exist. This is the single most common capacity mistake in the area,
and it is expensive in exactly the resource microVMs are least efficient with.

### Snapshot hygiene is a security step, not a polish step

Every restore starts from identical memory. Without intervention, a thousand
restores share entropy pool, PRNG state, clock, and machine identity — a
thousand sandboxes generating the same "random" values. After restore, as
part of the restore path:

```text
reseed entropy · resync the clock · inject fresh identity
rotate any guest credential · attach a fresh workspace · attach fresh policy
```

A snapshot is preparation, never tenant state.

---

## 12. Deep dive — reuse without residue

Warm pools invite the optimisation: tenant A finishes, clean the VM, hand it
to tenant B. For mutually untrusted tenants, **do not make cleaning the
security boundary**. Proving that an arbitrary compromised guest has been
sanitised is harder than destroying it.

```text
reuse       base image · boot snapshot · read-only package layers · host image cache
destroy     tenant writable memory · workspace overlay · credentials · process tree
```

> **Reuse immutable preparation. Destroy mutable tenant state.**

This keeps most of the latency benefit — the expensive parts are the immutable
ones — without a sanitisation proof nobody can give.

---

## 13. Checkpoint and restore: state is restored, authority is revalidated

Checkpoint semantics — what is in one, how it is published, fencing — belong
to [design #5](agent-checkpoint-replay-debugging.md) and the checkpoint deep
dive. The sandbox's part is narrower and has one rule.

A checkpoint captures workspace delta, optional VM state, sandbox metadata,
and a *reference* to the policy — never the grants. On restore:

```text
restore compute state
      ↓
validate the checkpoint's lineage (the #5 generation)
      ↓
mint a fresh sandbox generation
      ↓
re-evaluate the *current* capability policy
      ↓
issue fresh short-lived grants
```

Otherwise a six-hour-old checkpoint resurrects a network grant or a credential
that was revoked five hours ago, in a workspace nobody is watching.

> **State can be restored. Authority must be revalidated.**

---

## 14. Multi-tenancy and quotas

Two levels.

**Per tenant:** concurrent sandboxes, CPU-hours and memory-hours per day,
disk, egress bytes — and **startup rate**. A tenant that creates and destroys
sandboxes in a loop drains the warm pool without ever touching its
concurrency quota; the rate limit is what protects other tenants' p50.

**Fleet:** weighted fair scheduling across tenants and classes, with reserved
capacity for interactive production agents and release-blocking evaluation,
and preemption of exploratory batch.

**Security constraints override fairness.** A quarantined host or an
incompatible one is not made eligible because a tenant is under quota.

---

## 15. Observability

Latency, broken down by stage so the slow one is visible:

```text
time-to-ready p50/p99          by stage: placement · pull · restore/boot · attach
image cache hit rate           snapshot cache hit rate
warm pool depth vs arrival rate
```

Security, as signals rather than logs:

```text
egress bytes per sandbox, and by destination        denied egress attempts
grants issued / revoked per sandbox                  grants used after their sandbox died  ← must be zero
effect intents by class                              quarantine events, and time-to-quarantine
host integrity check failures                        residue checks on restored snapshots
```

The one to watch: **a grant used after its sandbox was destroyed.** It should
be structurally impossible; if the metric is ever non-zero, the
revoke-before-destroy ordering has broken somewhere, and that is the incident.

---

## 16. Security posture, stated once

```text
untrusted code           microVM per sandbox; no shared kernel across tenants
host                     no tenant authority; narrow materialisation grants
authority                none ambient; explicit, scoped, short-lived, revocable, bound to generation
network                  default deny; controlled DNS; private ranges denied; volume is a signal
secrets                  brokered per capability; never long-lived inside; endpoint over env var
effects                  through the gateway; the sandbox never holds the credential
snapshots                reseeded, re-identified, fresh policy on every restore
reuse                    immutable preparation only
destroy                  revoke first
compromise               quarantine, revoke, reimage — do not deliberate
```

---

## 17. Capacity estimation

From #1's numbers.

### Hosts

```text
10K concurrent × 2 vCPU / 4 GB          20K vCPU, 40 TB RAM
microVM overhead ~5% memory              ~42 TB
64 vCPU / 256 GB hosts, memory-bound     ~330 hosts
CPU oversubscribed 3–4× (agents idle)    CPU is not the constraint
```

### Warm pool

```text
peak arrivals ~10/s × ~5 s to replenish  ≈ 50, keep 100 for safety
100 × 4 GB                               0.4 TB — 1% of the fleet
```

Versus sizing to concurrency: 10K idle VMs, 40 TB. Two orders of magnitude.

### Image cache — the number that changes the shape

```text
distinct images in use    say 200, ~5 GB each
naive per-host cache      1 TB per host   ← does not fit next to workspaces
```

So per-host caches are populated by placement affinity (a host serves a
subset of images) and by lazy-pulling formats that start before the whole
image lands, backed by a shared content-addressed layer store. This is why
image locality is a *scheduling input* and not a cache tuning knob.

### Capability plane

```text
grants   a few per sandbox × 10 starts/s      trivial
proxy    10K sandboxes × mostly idle          tens of Gbit/s at worst; horizontal
audit    every grant, egress decision, effect  ~10⁷ events/day; append-only
```

### Egress budget

```text
10K × 500 MB max                            5 TB/day ceiling
observed, for coding agents                 mostly package installs; a fraction of that
```

The ceiling exists so a coordinated exfiltration is bounded, not because it
is expected.

---

## 18. Failure modes

| failure | handling |
|---|---|
| policy engine down | new sandboxes fail; none start with a default policy |
| broker down | grants fail; running sandboxes keep only unexpired grants |
| proxy down | egress fails closed; compute continues |
| host dies | sandboxes lost; #1's lease expiry handles the jobs; grants expire or are revoked by the reaper |
| image pull fails | placement retries elsewhere; time-to-ready breaches, no security effect |
| snapshot restore fails | boot clean; slower, not weaker |
| destroy stops mid-way | revocation already happened (first step); the reaper finishes the rest |
| stale client Exec | generation mismatch; rejected |
| suspected escape | quarantine the host; revoke; reimage |
| tenant loops create/destroy | startup-rate quota |
| snapshot restored without reseed | forbidden by the restore path; residue check catches a regression |

### The one worth dwelling on: the tempting fail-open

The policy engine has an outage. Ten thousand agents are waiting. The
proposal on the incident call: *start sandboxes with the last known good
policy, or a safe default, and reconcile later.*

Both are wrong, for different reasons. "Last known good" is per-tenant state
the host does not have and should not cache — a host that caches policy is a
host that decides authority. A "safe default" is a default; the moment it
exists, some code path will start a sandbox under it when it should not have,
and the audit log will say the sandbox had a policy.

The correct answer costs availability: sandboxes do not start. The design's
job is to make that outage short — a small, replicated, stateless policy
engine over a durable store — not to make it survivable by weakening the
boundary. **Fail closed on policy** is easy to say and hard to hold at 3 a.m.,
which is why it is written down.

### The second: destroy before revoke

A refactor reorders Destroy to stop the VM first, because it makes the
latency metric better. The VM is gone in 200 ms; the grants expire in five
minutes. In those five minutes a credential that was copied out during the
run — the sandbox was compromised, which is the premise — is valid, and
nothing is watching. The ordering is not an implementation detail; it is the
lifecycle boundary.

### What breaks first at 10×

Not the hosts — they scale by adding hosts. The image distribution problem
(100K cold starts want the same layers at once) and the audit log's write
rate. Content-addressed layer stores with host-level caching for the first;
partitioned append-only storage for the second.

---

## 19. Cost controls

```text
microVM overhead     ~5% memory; accepted for untrusted code
warm pool            sized to arrivals; ~1% of fleet
image cache          affinity + lazy pull; not a full cache per host
classes              interactive on reserved hosts; batch packed and preemptible
CPU oversubscription 3–4×, because agents idle; memory never
audit                append-only, tiered
```

---

## 20. Tradeoffs

| decision | chosen | given up | choose otherwise when |
|---|---|---|---|
| microVM per sandbox | real boundary | ~5% memory; pool machinery | first-party trusted code only |
| no ambient authority | bounded blast radius | every integration is explicit work | never |
| fail closed on policy | no sandbox without a policy | availability during control-plane outages | never |
| revoke before destroy | no credential outlives its sandbox | a slower destroy | never |
| destroy, never reuse across tenants | no sanitisation proof needed | some latency | single-tenant fleets |
| host holds no tenant authority | escape is survivable | more round trips to the planes | never |
| warm pool by arrivals | 1% of fleet idle | occasional p99 breach on a burst | latency SLO is tight and money is not |
| workload classes | interactive latency | packing efficiency | homogeneous workload |
| gVisor for some classes | density | compatibility surprises | known syscall profile, no nested containers |

---

## 21. Connections

Two of the other designs depend on this one in ways that are easy to miss.

**Evaluation's "no external effects" (#3) is not a policy in the evaluation
platform.** It is a sandbox capability policy: no effect grants, egress to
the mirror only, recorded responses for live services. The evaluation platform
requests it; this service is what makes it true.

**Replay's inertness and fork's record-only mode (#5)** are the same
mechanism: a replay executor is a sandbox with *no* capability plane
reachable; a record-only fork is a sandbox whose effect gateway logs intents
and returns synthetic success. Neither is enforced in agent code, because
agent code is the part that can be wrong.

And the instrumentation rule from #1 §13 lives here: the host records what
the sandbox did — commands, egress, effects — out of band. The agent does not
observe itself being recorded, so the measurement does not change the
trajectory.

---

## 22. Telling it in 45 minutes

```text
0–5     four boundaries; adversarial by proxy; scope; what the deep dive already owns
5–10    three planes and what each is ignorant of; the blast-radius chain
10–16   no ambient authority: network, secrets, effects; grants bound to generation
16–21   lifecycle; revoke before destroy; generation on identity
21–27   resource isolation as security; workload classes
27–33   two-second starts: placement for time-to-ready; pool by arrivals; snapshot hygiene
33–37   reuse without residue; restore revalidates authority
37–41   capacity: hosts, pool, the image-cache number
41–45   the fail-open story; build first
```

The opening mistake: fifteen minutes on gVisor versus Firecracker. That is
one row of one table, and the deep dive already has it.

---

## 23. Questions the design has to survive

1. **Why not containers?** Shared kernel; a filter narrow enough to be a
   boundary breaks the agent, one wide enough for the agent is not a
   boundary. MicroVM for untrusted code; the deep dive has the full argument.
2. **The VM is escaped. Then what?** The host holds no tenant authority; the
   planes do not trust hosts; every grant is bound to a generation and
   revocable. Quarantine, revoke, reimage — without deliberating. §8.
3. **How does an agent `pip install` under default deny?** Through the egress
   proxy to a package mirror, with controlled DNS. Deny that works is deny
   nobody works around. §9.
4. **Where do credentials live?** Nowhere inside. A broker performs the
   operation or mints a minutes-long capability bound to the sandbox. The
   strongest credential is the one that never entered. §9.
5. **p50 start is 45 seconds. What first?** Look at the stage breakdown; it is
   the image pull. Image locality as a placement input, lazy pull, then
   snapshot restore. None of it touches the boundary. §11.
6. **Snapshot restore gives 100 ms starts. What did you break?** Entropy,
   clock, identity — a thousand sandboxes with the same randomness. Reseed in
   the restore path. §11.
7. **Reuse a warm VM across tenants?** No. Reuse immutable preparation;
   destroy mutable state. Cleaning is not a boundary anyone can prove. §12.
8. **Restore a six-hour-old checkpoint. Do its grants come back?** No. State
   is restored; authority is re-evaluated against current policy and
   re-issued fresh. §13.
9. **Policy engine is down and ten thousand jobs are queued.** They wait.
   Fail closed; make the outage short, not survivable-by-default. §18.
10. **At 100×?** Image distribution and the audit log, not hosts. Content-
    addressed layers with host caching; partitioned append-only audit. §18.

---

## 24. What to build first

The **capability plane's identity binding** — grants bound to
`(sandbox_id, generation)`, indexed for one-operation revocation — and the
**lifecycle ordering** that revokes before it destroys. The isolation
technology can be swapped later; a fleet that started with ambient authority
cannot be retrofitted to no-ambient-authority without touching every
integration that came to depend on it, and by then something will.
