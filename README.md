# System design notes

Notes on designing AI/ML and agent infrastructure, written to understand it
rather than to summarise it. One document per system, each worked through from
requirements to failure modes, with the numbers measured where they could be
and stated as guesses where they could not.

The subject is the stack underneath long-running AI agents: what runs them,
what serves the model to them, how they are evaluated, how they are isolated,
and how their state is saved and recovered. The designs come from building
pieces of that stack for [AgentSeism](https://github.com/lyr-ai/agentseism)
and from the questions that came up doing so.

## The series

Six designs that compose. The first is the whole picture; #2–#5 are deep
dives into one box of it, which is why they are worth reading in order — each
one reuses the vocabulary the previous one established. #6 is the branch
upstream of all of them: how a model gets made before #2 serves it, and where
the RL loop turns the other five into a training system.

| # | design | state | it deep-dives |
|---|---|---|---|
| 1 | [Agent Execution Platform](designs/agent-execution-platform.md) | drafted | — the whole system |
| 2 | [Multi-tenant LLM inference platform](designs/llm-inference-platform.md) | drafted | the Model Service box |
| 3 | [Agent evaluation and experimentation platform](designs/agent-evaluation-platform.md) | drafted | what runs on top of #1 and #2 |
| 4 | [Secure sandbox service for untrusted code](designs/secure-sandbox-service.md) | drafted | the Sandbox box, as a service #1 calls |
| 5 | [Agent checkpoint, replay and debugging](designs/agent-checkpoint-replay-debugging.md) | drafted | the Checkpoint + Event boxes, and what runs on them |
| 6 | [Post-training and fine-tuning platform](designs/post-training-platform.md) | drafted | what happens before a model reaches #2 |

`state` is one of `outline` · `drafted` · `condensed` — the last meaning a
two-page version exists **and** the design can be explained from it without
the full notes. Having the page is not the state; being able to give the
explanation is, which is why #3 has a condensed page and is still `drafted`.

## Deep dives

T-shaped, deliberately. Each design gets its architecture straight, then four or
five sections go deep — the ones where the system is genuinely hard, or where
the obvious answer is wrong. Everything else stops at "knows the tradeoff",
because writing all twenty sections to full depth is writing a distributed
systems textbook, and understanding does not scale with the page count.

The bar for done: point at any core box and ask *why*, *what if it fails*,
*what at 100×*, and the answer runs for five to ten minutes without notes. At
that point stop polishing and start the next design.

| deep dive | of | state |
|---|---|---|
| [Scheduler, lease, failure recovery](deep-dives/scheduler-lease-recovery.md) | #1 §8 §9 §18 | drafted |
| [Checkpoint and resume](deep-dives/checkpoint-resume.md) | #1 §12 | drafted |
| [Sandbox and isolation](deep-dives/sandbox-isolation.md) | #1 §10 §11 | drafted |
| Control/data plane and the failure model | #1 §6 §18 | — |
| Capacity with a shared GPU pool | #1 §8 §17 | covered by #6 §9 |

## Condensed versions

The step from `drafted` to `condensed`: the same design compressed to what
would be said explaining it to someone in one sitting, with section pointers
back to the full text. Two pages, no new content. If it cannot be told from
the condensed page, the full text has not been understood yet.

| condensed | as a talk | of |
|---|---|---|
| [bullets](condensed/agent-evaluation-platform.md) | [spoken, with transitions](condensed/agent-evaluation-platform-talk.md) | #3 |

The bullets are what goes on the board; the talk is what gets said, segment
by segment, with the transition sentence between segments — which is where an
explanation actually stalls.

## Document shape

Fixed by the first document and followed by the rest, so that revising is
scanning a column rather than re-reading prose. See
[TEMPLATE.md](TEMPLATE.md).

The two sections that do the most work are the last ones. **Questions the
design has to survive** is where most write-ups stop too early: almost
everyone works out the happy path and has nothing for the first hard push.
**What to build first** forces a claim about which part is load-bearing, which
is the question worth being able to answer about any system.

## Where the numbers come from

Several designs lean on measurements rather than estimates: decode and prefill
rates on a 27B FP8 model under vLLM, the cold/warm prefix-cache gap on a 53K
token prompt, run-length spread on a single coding task, how often an
identically reconstructed agent state reproduces its next actions, and how
often a model call had to be re-sampled because of transport failures. They
come from the experiments in
[AgentSeism](https://github.com/lyr-ai/agentseism), and the write-ups of
those experiments are at [lyr-ai.github.io](https://lyr-ai.github.io).
