# Deep dive — Sandbox and Isolation

> Belongs to [System Design #1](../designs/agent-execution-platform.md), §10 §11.
> Completes the triangle: run the agent, serve the model, **isolate the
> execution**.

Four questions, and nothing else. Everything here exists to answer one of them.

```text
1  Why isn't a container enough?
2  gVisor or Firecracker — how do you choose?
3  How do you isolate filesystem, network and credentials for arbitrary code?
4  Strong isolation and a two-second start — how do you get both?
```

**The point.** Naming a threat model and designing to it, rather than reciting
three container technologies. The fourth question is where most designs
collapse, because they assume isolation and speed trade against
each other. Mostly they do not.

---

## 1. Why a container is not enough

A container is a process with namespaces, cgroups and a seccomp filter. **The
kernel is shared.** Everything else follows from that sentence.

### The threat model

Not "buggy code". The code was written by a model following instructions from a
stranger, and the stranger may have written those instructions specifically to
get out. Call it **adversarial by proxy**: no human attacker is present, but the
input distribution includes whatever an attacker chose to put in a GitHub issue.

So the question is not *is this likely* but *what is the blast radius when it
happens*. With a shared kernel, one kernel bug reaches every tenant on the host.

### Why seccomp does not save it here

The usual mitigation is a tight syscall filter. That works when you know what the
workload does. A coding agent legitimately needs to:

```text
compile          fork and exec freely      install packages
run test suites  write and delete files    sometimes run containers itself
```

which is most of the syscall surface. **A filter narrow enough to be a real
boundary would break the agent; a filter wide enough for the agent is not a real
boundary.** That is the argument, and it is specific to this workload rather than
a general complaint about containers.

Container escapes are recurring rather than hypothetical — the runc and kernel
LPE history is long enough that "unlikely" is not a design position.

### The second-order problem

Even without escape, a shared kernel means shared page cache, shared scheduler,
shared kernel data structures. Tenants interfere and can observe each other's
timing. For a platform whose whole value proposition is running other people's
code, that is a poor starting position.

---

## 2. gVisor or Firecracker

|  | **gVisor** | **Firecracker** |
|---|---|---|
| boundary | a userspace kernel (Sentry) implements syscalls | hardware virtualisation, KVM |
| host surface | the Sentry plus a small host syscall set | the VMM plus KVM |
| overhead | 10–30% on syscall-heavy work, worst on file I/O | ~5% memory; near-native CPU |
| startup | ~150–200 ms | ~125 ms VMM + guest boot |
| compatibility | **partial** — unimplemented syscalls exist | full Linux |
| nested containers | awkward | just works |

### The decision rule

Ask three questions in this order:

```text
1  What is the threat model?          untrusted → a real boundary is required
2  What is the syscall profile?       heavy file I/O → gVisor's tax is large
3  What compatibility is needed?      arbitrary toolchains → gVisor will bite
```

**For coding agents: Firecracker-class microVMs.** The deciding factor is not
security — both are defensible — it is **compatibility**.

An unimplemented syscall does not fail cleanly. The agent sees a strange error,
concludes its own code is wrong, and spends ten steps and real money debugging
the sandbox instead of the task. That failure is expensive, hard to attribute,
and it recurs for every new toolchain a user brings.

And agents increasingly want to run containers themselves — a test harness, a
service under test. Inside a microVM that is just Docker on Linux. Inside gVisor
it is a research project.

### When gVisor is the better answer

Not never:

```text
first-party workloads with a known syscall profile
short, compute-bound, little file I/O
density matters more than compatibility
no nested virtualisation needed
```

**Do not claim one is universally correct.** Naming the axis that decides —
compatibility against density — is the point; a side picked without one is not.

---

## 3. The three surfaces beyond the kernel

The kernel boundary stops escape. It does nothing about a sandbox that simply
*asks* for things.

### Filesystem

```text
read-only base image  +  per-sandbox writable overlay
no host mounts, ever
ephemeral by default; the workspace is the only thing that survives
```

The overlay is what makes the checkpoint story from
[deep dive #2](checkpoint-resume.md) cheap: the writable layer *is* the
workspace delta.

### Network

**Default deny.** Then make the denial survivable, because a control everyone
works around is not a control:

```text
sandbox → egress proxy → allowlist
        → package mirror              so "no internet" ≠ "cannot pip install"
        → controlled DNS              or DNS becomes the exfiltration channel
```

DNS is the one people forget. An allowlisted HTTP proxy with an open resolver
leaks data one subdomain at a time.

Treat **egress volume as a signal**, not only egress destination. A job that
reads the repository and posts 400 MB somewhere allowed is still worth an alert.

### Credentials

```text
short-lived, scoped, minted per job
never a long-lived key
never written to the sandbox filesystem
```

Prefer a metadata endpoint the sandbox queries over injecting a token into the
environment, because an endpoint can be revoked the moment the job ends and an
environment variable cannot be taken back.

**Assume anything inside the sandbox is exfiltrated.** Design so that the worst
case is a scoped token with minutes of life, not an org-wide key.

### And the strongest control is architectural

From [deep dive #1 §8](scheduler-lease-recovery.md): irreversible actions go
through a **guarded effect service**, so the sandbox never holds credentials for
them at all.

> The best way to survive a stolen credential is not to have given the sandbox
> one.

---

## 4. Strong isolation *and* a two-second start

The budget from [design #1](../designs/agent-execution-platform.md):

```text
p50 < 2 s     p99 < 10 s
```

### Where the time actually goes

```text
VMM process start          ~125 ms
guest kernel boot          ~100–500 ms
image pull                 seconds to minutes      ← dominant
workspace materialisation  depends on repo size
```

**The dominant term has nothing to do with isolation.** Which is why the usual
framing — strong isolation versus fast startup — is largely a false trade. The
slow part is pulling gigabytes and booting a kernel, and both are removable
without touching the security boundary.

### Three levers, in order of what they buy

**Image caching and locality.** Pulling a multi-gigabyte image turns a
two-second start into two minutes. Cache per host and make **image locality a
routing input** — a worker with the image already resident beats a worker with
four times the free capacity ([design #1 §11](../designs/agent-execution-platform.md)).
Lazy-pulling formats that start before the whole image lands help further.

**Snapshot restore.** Boot one VM, let the kernel and runtime initialise,
snapshot it, and restore that snapshot per sandbox. Restore skips boot entirely
and lands in the low hundreds of milliseconds.

> **The security caveat that must be said out loud:** every restore starts from
> *identical* memory. Entropy pools, PRNG state, clocks and any per-instance
> identity are the same in all of them. Restoring one snapshot a thousand times
> without reseeding gives a thousand sandboxes generating the same "random"
> values — a real vulnerability, not a curiosity. Reseed entropy, resync the
> clock and inject identity **after** restore, as part of the restore path
> rather than as a later step.

**Warm pools.** Keep pre-booted, pre-restored VMs waiting. Allocation becomes
attach-the-workspace rather than start-a-machine.

Size the pool to **arrival rate**, not concurrency:

```text
100K jobs/day  ≈ 1.2 arrivals/s average, perhaps 10/s at peak
a pool of ~50–100 absorbs that

sizing to 10,000 concurrent instead  →  two orders of magnitude of idle VMs
```

This is the single most common capacity mistake in this area, and it is
expensive in exactly the resource microVMs are least efficient with.

### What you actually trade

| lever | costs | weakens isolation? |
|---|---|---|
| image cache | disk per host, routing complexity | no |
| snapshot restore | snapshot management, **entropy handling** | only if the caveat is ignored |
| warm pool | idle capacity proportional to arrival rate | no |
| smaller base image | maintenance, user friction | no |
| dropping to containers | — | **yes, entirely** |

Only the last row is a genuine isolation trade, and it is the one not to take
for untrusted code. Everything above it is engineering effort converted into
latency.

> **You do not trade isolation for startup time. You pay for startup separately,
> with caching, snapshots and pooling, and keep the boundary.**

---

## 5. Questions to check understanding

1. What is the threat model for a coding agent's sandbox? Say it in one sentence.
2. Why can seccomp not do the job here, specifically?
3. gVisor or Firecracker for this workload, and what decides it?
4. Give a case where gVisor is the better choice.
5. An agent needs to run Docker for its integration tests. What changes?
6. Default-deny egress breaks `pip install`. Now what?
7. Why is DNS a hole in an HTTP allowlist?
8. Where do credentials live, and what is the blast radius when they leak?
9. p50 sandbox start is 45 s. Where is the time, and what do you fix first?
10. Snapshot restore gives you 100 ms starts. What did you just break?
11. How big is the warm pool, and what number is it derived from?
12. Which of your startup optimisations weakens the security boundary?
