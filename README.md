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
| 2 | Multi-tenant GPU inference service | — | the Model Service box |
| 3 | Large-scale agent evaluation platform | — | what runs on top of #1 |
| 4 | Secure sandbox service for untrusted code | — | the Sandbox box |
| 5 | Agent checkpoint / replay / debugging | — | the Checkpoint + Event boxes |

`state` is one of `outline` · `drafted` · `whiteboard` — the last meaning it can
be delivered in 45 minutes without the notes, which is the only state that
counts on the day.

## Document shape

Fixed by the first document and followed by the rest, so that revision is
scanning a column rather than re-reading prose. See
[TEMPLATE.md](TEMPLATE.md).

The two sections that do the most work are the last ones. **Expected follow-ups**
is where most preparation stops too early: almost everyone rehearses the happy
path and falls apart on the first push. **What to build first** forces a claim
about which part is load-bearing, which is the question a senior interviewer is
actually asking.
