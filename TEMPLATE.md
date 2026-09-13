# System Design #N — <Title>

> **<The question this design answers, in one sentence.>**

<Two or three sentences of the concrete case. Then the framing sentence: what
kind of system this really is, so the answer does not drift into the wrong one.>

## 1. Problem statement
What the client submits, what the platform does, what it must support. Keep it
in lists — this section is for orienting, not arguing.

## 2. Clarify before designing
A table: question → what it changes in the architecture. Only questions whose
answers change the design. Close with the scope you are assuming, stated out
loud, including what you are explicitly *not* designing.

## 3. Non-functional requirements
Isolation · reliability · scale · latency · observability, with numbers. Each
one should imply something later in the document; if it does not, cut it.

## 4. Data model
The three to five entities, and why the boundaries between them are where they
are. Boundaries are the content here, not fields.

## 5. High-level architecture
One diagram, drawn in the order you would draw it live. Name each box in one
line. Cross-cutting services listed separately.

## 6. Control plane vs data plane
Who runs user code and who decides what runs. State the blast-radius argument.

## 7. Lifecycle
The happy path as a state machine, then the crash path.

## 8–16. Deep dives
One section per place the system is genuinely hard. For each: the options, the
tradeoff, the choice, and the reason. A choice without a named tradeoff reads as
inexperience.

## 17. Capacity estimation
Show the arithmetic. Find the one number that changes the shape of the design
and say it plainly.

## 18. Failure modes
A table, plus one failure worth dwelling on — ideally one you have actually
seen, which is what separates a lesson from a summary.

## 19–20. Cost and tradeoffs
Tradeoff table: decision · chosen · given up · when you would choose otherwise.

## 21. Connections
Where this touches your own work, arriving as a consequence of the design rather
than as a detour.

## 22. Telling it in one sitting
The order to explain it in, rough time per part, and the one opening mistake
to avoid. If it cannot be told in 45 minutes, it is not understood yet.

## 23. Questions the design has to survive
Ten questions, with answers. Writing them is where the gaps show up.

## 24. What to build first
One or two pieces, with the reason: what is painful to change later.
