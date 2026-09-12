# System Design #2 — Multi-tenant LLM Inference Platform

> **Design a platform that serves multiple open-weight models to many internal
> agent and application workloads from a shared pool of GPUs.**

```text
thousands of concurrent requests     streaming responses
long context                         multiple model sizes
multi-tenant quotas                  low TTFT
high throughput                      GPU efficiency
autoscaling                          failure recovery
cost control                         OpenAI-compatible API
```

This is the Model Service box that [System Design #1](agent-execution-platform.md)
deliberately declared out of scope. The two compose:

```text
Agent Runtime
     │
     ▼
Model Gateway
     │
     ├── routing
     ├── quota
     ├── admission control
     └── backpressure
     │
     ▼
Inference Scheduler
     │
     ▼
GPU Replica Pool
     │
     ├── prefill
     ├── decode
     ├── batching
     └── KV cache
```

**Part 1** covers the workload model and why this is not a web serving problem.
Everything after — routing, autoscaling, fairness, cost — depends on getting
this part right, and most weak answers go wrong here rather than later.

---

## 1. Clarify

| question | what it changes |
|---|---|
| Open-weight self-hosted, or a vendor API behind a gateway? | if vendor, this is a proxy design and GPU scheduling disappears |
| How many models, and how large? | one model per replica, or multiple resident; LoRA versus full weights |
| Context length? | KV cache sizing, which sets replica capacity (§7) |
| Interactive or batch? | TTFT matters or it does not, and that changes the scheduler |
| Is the caller an agent? | agent traffic has a very specific shape (§8) |
| Strict tenant isolation, or fair sharing? | separate replicas versus shared batches |
| Latency SLO, and on which metric? | TTFT and TPOT are different problems |

**Scope to state:** self-hosted open-weight models, a handful of sizes,
interactive traffic dominated by long-context agent workloads, fair sharing with
per-tenant quotas rather than dedicated hardware.

---

## 2. What a request actually costs

The first thing to establish, because every later decision follows from it: an
LLM request is **not a unit of work**. Its cost has two components with
different shapes.

```text
cost(request) = prefill(prompt_tokens) + decode(output_tokens)
```

- `prompt_tokens` is known at admission.
- **`output_tokens` is not known until the request finishes.**

That second line is the one that breaks every scheduling intuition borrowed from
web serving. You cannot bin-pack, queue by expected cost, or set a meaningful
timeout for work whose size is only revealed by doing it.

Measured on one deployment — Qwen3.6-27B-FP8, single A100 80GB, vLLM 0.28:

```text
2,000 output tokens   →  41.0 s      ≈ 49 tokens/s single stream
  600 output tokens   →  12.7 s      ≈ 47 tokens/s
```

Decode rate is stable. **Total request time is therefore almost entirely a
function of how much the model decides to say**, which nothing upstream knows in
advance.

---

## 3. Prefill and decode are two different machines

The single most important fact in this design.

|  | **prefill** | **decode** |
|---|---|---|
| what it does | process the whole prompt at once | produce one token, then the next |
| parallelism | all prompt tokens in parallel | strictly sequential per sequence |
| FLOPs | O(prompt length) | O(1) per token |
| bound by | **compute** | **memory bandwidth** |
| happens | once | thousands of times |
| batching | already saturated | nearly free throughput |

**Why decode is bandwidth-bound.** To produce one token the GPU must read the
entire weight matrix from HBM. With ~31 GB of FP8 weights, one decode step moves
31 GB regardless of whether it is generating for one sequence or for sixty-four.
At batch 1 that bandwidth produces a single token; at batch 64 the same read
produces sixty-four.

> **Decode at batch size 1 wastes almost all of the GPU.** Batching is not a
> throughput optimisation in LLM serving — it is the difference between using
> the hardware and not.

**Why prefill is different.** Prefill is a dense matrix multiply over the whole
prompt and is already compute-saturated. Batching prefills does not multiply
throughput the way batching decode does; two large prefills mostly compete.

This asymmetry is why the two phases want opposite scheduling policies, and why
running them on the same device at the same time is the central tension of the
whole design.

### The measurement that makes it concrete

Same endpoint, a 53,348-token prompt, three consecutive identical requests:

```text
cold   53,348 prompt tokens  →  13.0 s
warm                          →   0.9 s
warm                          →   0.6 s
```

Roughly **20×**, and it is entirely prefill: the decode portion was identical in
all three. The second and third requests skipped the prompt because the KV cache
still held it — which is the whole argument for prefix caching (§8) and the
reason routing is a first-class concern rather than a load-balancer setting.

---

## 4. Why this is not a web serving problem

Five differences, each of which invalidates a standard technique.

**1. Request cost is unknown and unbounded.** Web requests are approximately
constant work. Here the distribution is wide and the tail is the point — a
request may emit 20 tokens or 8,000. Queueing disciplines that assume knowable
service time do not apply.

**2. A request occupies memory for its entire lifetime.** The KV cache for a
sequence is allocated at admission and grows with every token until the request
ends. The scheduling unit is *a slot held for tens of seconds*, closer to a VM
than to an HTTP request. A replica with 64 slots serving 60-second requests
admits roughly one request per second, whatever its FLOPs say.

**3. The bottleneck is memory, not CPU or FLOPs.** Capacity is set by how many
tokens of KV cache fit alongside the weights (§7). Adding compute does not add
capacity.

**4. Latency has two independent metrics.** They fail for different reasons and
one can be fine while the other is terrible:

```text
TTFT   time to first token    = queueing + prefill
TPOT   time per output token  = decode batch contention
```

A large prefill in the batch hurts *everyone's* TPOT. A deep queue hurts only
TTFT. Reporting "p99 latency" over whole requests hides both, because it is
dominated by output length, which is a property of the request rather than of
the service.

**5. Requests are preemptible mid-flight.** Unlike an HTTP handler, a sequence
can be evicted and resumed — recompute its KV cache, or swap it out. That makes
admission less dangerous than it looks and gives the scheduler a lever no web
tier has.

### Head-of-line blocking is physical here

The decode loop advances every sequence in the batch one token per step. If a
large prefill is scheduled into that loop, every other sequence stalls for its
duration.

```text
decode step, decode step, [ 30k-token prefill ], decode step, ...
                           ↑
                  every sequence in the batch waits
```

This is why **chunked prefill** exists: split a long prompt into pieces and
interleave them with decode steps, trading a little TTFT for the long request to
protect TPOT for everyone else. It is a scheduling decision inside the engine,
and being able to name it is a strong signal.

---

## 5. Continuous batching

Static batching — collect N requests, run them together, return together — is
wrong here for a specific reason: sequences finish at different times, so the
batch runs at the speed of its longest member while finished slots sit idle.

**Continuous batching** treats the batch as a set that changes every step: when a
sequence emits its end token its slot is freed immediately and a waiting request
takes it at the next step.

```text
step k    [ A B C D ]
step k+1  [ A B C D ]      B finishes
step k+2  [ A E C D ]      E admitted into B's slot
```

Two consequences worth stating:

- **Admission happens continuously**, not at request arrival. The queue is
  drained at every decode step, so queue depth translates directly into TTFT.
- **`max_num_seqs` is the batch width**, and it is a memory decision rather than
  a throughput one — each concurrent sequence needs its own KV cache.

---

## 6. The memory budget, which is the capacity model

Everything a replica can do is decided by one subtraction.

```text
HBM
  − model weights
  − activations and runtime overhead
  = KV cache pool
```

KV cache per token:

```text
2 (K and V) × layers × kv_heads × head_dim × bytes_per_element
```

Measured anchor from one deployment:

```text
Qwen3.6-27B-FP8 weights          ≈ 30.9 GB
A40 48 GB     → ~17 GB for KV cache and runtime
A100 80 GB    → ~49 GB
```

The consequence that decides hardware:

```text
max_model_len = 32,768    one full-length sequence fits easily on 48 GB
max_model_len = 131,072   one full-length sequence needs roughly 4×
```

vLLM refuses to start if the KV pool cannot hold a single sequence of
`max_model_len`, so **context length is a hardware decision, not a
configuration flag**. Raising it from 32k to 128k on the same card does not make
long requests slower; it makes the server not start.

> Capacity in this system is *tokens of KV cache*, not requests per second.
> Every other number is derived from it.

### The tension to name

```text
long max_model_len   → fewer concurrent sequences
large batch          → higher throughput, more KV pressure
both at once         → preemption, or admission rejection
```

Three knobs — `max_model_len`, `max_num_seqs`, `gpu_memory_utilization` — all
spending the same budget. One measured deployment moved `max_num_seqs` from 8 to
4 precisely to buy headroom when the context window was widened.

---

## 7. Where Part 2 picks up

With the workload model established, the remaining questions are all downstream
of it:

```text
routing              prefix affinity, and why round-robin is expensive
prefix caching       when the 20× applies and when it does not
replica sizing       one model per GPU, tensor parallel, or multiple models
autoscaling          scaling on what signal, given cold start is minutes
multi-tenancy        fairness over what unit, when requests are not equal
failure handling     a replica dies holding 60 sequences of state
cost                 tokens, GPU-hours, and which one the bill is in
agent traffic        the specific shape it has, and why it is the good case
```

The order matters: routing and autoscaling both reduce to the memory budget and
the prefill/decode asymmetry, and answering them without §3 and §6 produces
plausible-sounding designs that do not survive arithmetic.
