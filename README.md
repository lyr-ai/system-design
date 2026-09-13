# System design notes

Preparation for system design interviews, weighted toward AI/ML and agent
infrastructure. One document per topic, written rather than scaffolded.

## The series

Five designs that compose. The first is the whole picture; the rest are deep
dives into one box of it, which is why they are worth doing in order — each one
reuses the vocabulary the previous one established.

| # | design | state | it deep-dives |
|---|---|---|---|
| 1 | [Agent Execution Platform](designs/agent-execution-platform.md) | drafted | — the whole system |
| 2 | [Multi-tenant LLM inference platform](designs/llm-inference-platform.md) | drafted | the Model Service box |
| 3 | [Agent evaluation and experimentation platform](designs/agent-evaluation-platform.md) | drafted | what runs on top of #1 and #2 |
| 4 | Secure sandbox service for untrusted code | — | the Sandbox box |
| 5 | Agent checkpoint / replay / debugging | — | the Checkpoint + Event boxes |

`state` is one of `outline` · `drafted` · `whiteboard` — the last meaning it can
be delivered in 45 minutes without the notes, which is the only state that
counts on the day.

## Deep dives

T-shaped, deliberately. Each design gets its architecture straight, then four or
five sections go deep — the ones most likely to be pushed on. Everything else
stops at "two or three minutes and knows the tradeoff", because writing all
twenty sections to this depth is writing a distributed systems textbook, and the
interview return does not scale with the page count.

The bar for done: the interviewer points at any core box and asks *why*, *what
if it fails*, *what at 100×*, and the answer runs five to ten minutes without
notes. At that point stop polishing and start the next design.

| deep dive | of | state |
|---|---|---|
| [Scheduler, lease, failure recovery](deep-dives/scheduler-lease-recovery.md) | #1 §8 §9 §18 | drafted |
| [Checkpoint and resume](deep-dives/checkpoint-resume.md) | #1 §12 | drafted |
| [Sandbox and isolation](deep-dives/sandbox-isolation.md) | #1 §10 §11 | drafted |
| Control/data plane and the failure model | #1 §6 §18 | — |
| Capacity with a shared GPU pool | #1 §8 §17 | — |

## Whiteboard skeletons

The step from `drafted` to `whiteboard`: the same design compressed to what
gets said unprompted in 45 minutes, with section pointers back to the full
text. Two pages, no new content.

| skeleton | walkthrough | of |
|---|---|---|
| [bullets](whiteboard/agent-evaluation-platform.md) | [spoken, with transitions](whiteboard/agent-evaluation-platform-walkthrough.md) | #3 |

The skeleton is what goes on the board; the walkthrough is what gets said,
segment by segment, with the transition sentence between segments — which is
where a delivery actually stalls.

## Document shape

Fixed by the first document and followed by the rest, so that revision is
scanning a column rather than re-reading prose. See
[TEMPLATE.md](TEMPLATE.md).

The two sections that do the most work are the last ones. **Expected follow-ups**
is where most preparation stops too early: almost everyone rehearses the happy
path and falls apart on the first push. **What to build first** forces a claim
about which part is load-bearing, which is the question a senior interviewer is
actually asking.
