# Memory Consistency Models: Problem, History, and Landscape

### A First Draft of the Term Paper Introduction and Survey

**Course:** Computer Architecture
**Draft stage:** Section 1 of an expected ~150-page term paper (problem introduction, problem statement, historical development, and literature survey)
**Scope of this draft:** Foundations, motivation, history, breadth-and-depth of the field, and an annotated literature review, all anchored to the prescribed course texts and extended with primary research sources.

---

> **[FORMATTING NOTE — for the LaTeX pass, delete before submission]**
> Throughout this document, non-text components that belong in the final paper are flagged inline in the form
> **[FIGURE n]**, **[TABLE n]**, **[GRAPH n]**, or **[DIAGRAM n]**, each with a caption and a description of what it should contain. These are placeholders: the surrounding prose is written so that the argument is complete even before the visuals are drawn, but the visuals are part of the intended composition and should be produced during typesetting. Citations are given inline as author–year with a resolvable link in the reference list at the end; convert these to `\cite{}` keys against a `.bib` file during the LaTeX pass. Course-text cross-references (e.g., "Sarangi, *Basic Computer Architecture*, §12.4.3") are given with section/page numbers keyed to the specific editions listed in the references.

---

## Abstract

On a single processor, a program means what its text says it means: instructions appear to take effect one after another, in the order written. Decades of microarchitectural cleverness — pipelining, out-of-order execution, write buffers, caches — are carefully hidden behind this illusion, so that the programmer never has to know about them. The moment two or more processors share memory, the illusion breaks. Each core still reorders and buffers its own operations for speed, but now one core's reordering becomes visible to another, and programs can observe outcomes that are impossible under any straightforward interleaving of their instructions. A **memory consistency model** is the contract that resolves this: it is the precise, architecturally visible specification of which orderings of memory reads and writes a multiprocessor is permitted to exhibit, and therefore which values a read is allowed to return.

This paper — the first installment of a longer term paper — establishes the problem, its history, and its landscape. We begin from the uniprocessor "sequential" contract familiar from pipelined-processor design and show, through small concurrent programs called *litmus tests*, exactly how and why shared memory violates naïve intuition. We draw the sharp and frequently confused line between **coherence** (the well-behavedness of a single memory location) and **consistency** (the ordering of accesses across locations). We then trace the intellectual history of the field: from the pipelined-computer roots of instruction reordering, through Lamport's 1979 definition of **sequential consistency**, the 1980s proliferation of relaxed hardware models, the 1990s programmer-centric synthesis and release consistency, the 2000s move to *language-level* memory models in Java and C++, and the 2010s program of rigorous, tool-backed models for x86, ARM, and IBM POWER. Finally, we map the breadth of the subject — hardware models, language models, specification styles, and verification tooling — and set out the research questions and experimental plan that the remainder of the term paper will pursue. The treatment is deliberately anchored to the prescribed course texts (Sarangi's two volumes, Hennessy & Patterson, and Kogge on pipelining) and extended with the primary literature so that the reader can move from the course's framing into the research frontier without a discontinuity.

---

## Table of Contents (this draft)

1. Introduction
2. Background: From Uniprocessor Order to Multiprocessor Surprise
3. The Problem Statement
4. Sequential Consistency: The Reference Model
5. A History of Memory Consistency Models
6. The Landscape of Hardware Memory Models (Breadth)
7. Language-Level Memory Models
8. Specifying and Verifying Memory Models (Depth)
9. Annotated Literature Survey
10. Discussion: Depth, Breadth, and Open Problems
11. Plan for the Remainder of the Term Paper
12. Conclusion
— References

---

## 1. Introduction

### 1.1 The setting: why shared memory became unavoidable

For roughly the first four decades of commercial computing, the dominant way to make programs run faster was to make a single processor faster: raise the clock frequency, and do more work per cycle by overlapping the execution of instructions. The course's pipelining material (Sarangi, *Basic Computer Architecture*, Ch. 10; and, historically, Kogge, *The Architecture of Pipelined Computers*) is precisely the story of that overlap — of dividing instruction processing into stages so that many instructions are in flight at once, and of the hazard-handling machinery (interlocks, forwarding, branch handling) needed to preserve correctness while doing so. Sarangi's advanced volume (*Next-Gen Computer Architecture*, Ch. 2) continues the arc into out-of-order pipelines, where instructions are deliberately executed in an order different from the program's, subject only to respecting true data and control dependences.

Two physical limits ended the free ride. Frequency scaling stalled because power dissipation grows steeply with clock speed, and the returns from making a single core wider and deeper diminished. As Sarangi puts it in the motivation for the advanced text, after roughly 2005–2010 single-core performance saturated, and the extra transistors that Moore's law kept delivering were redirected from making one core more complex to placing *many* cores on one chip (Sarangi, *Next-Gen*, Ch. 1; *Basic*, §12.1 on Moore's law). Hennessy & Patterson frame the same inflection as the industry-wide turn to thread-level and request-level parallelism (H&P, 5th ed., Ch. 5). The multicore era was therefore not a design preference but a consequence: parallel hardware became the only way to keep converting transistors into performance.

Multicore hardware, however, only delivers performance if software can use many cores at once, and the dominant model for doing so is **shared memory**: several cores read and write a single common address space, communicating implicitly through loads and stores rather than through explicit messages (Sarangi, *Basic*, §12.2.2, "Shared Memory vs Message Passing"; *Next-Gen*, §9.1.1). Shared memory is attractive because it is a natural extension of the uniprocessor programming model — the same pointers, the same arrays, the same variables — and because it avoids the bookkeeping of message passing. But this convenience conceals a deep problem, and that problem is the subject of this paper.

> **[FIGURE 1]** — *Caption:* "The transition from single-core to multicore, and the emergence of the consistency problem." *Description:* A two-panel schematic. Left panel: a single core with a pipeline and a private cache in front of a monolithic memory, labeled "one apparent order of memory operations." Right panel: four cores, each with its own store buffer and private L1 cache, connected through an interconnect/coherence layer to shared memory, labeled "each core reorders and buffers independently — orderings become mutually visible." This figure motivates the whole paper and should appear early.

### 1.2 The problem in one paragraph

Consider two cores. Core 1 writes the value 1 to a shared variable `X` and then reads a shared variable `Y`. Core 2 writes 1 to `Y` and then reads `X`. Both variables start at 0. Intuitively — if we imagine the four operations interleaved in *some* single global order — at least one of the two reads should see the value 1, because by the time both reads have happened, both writes have "occurred." Yet on essentially every mainstream processor you can buy, including the x86 laptop this draft is being written on, both reads can return 0 in the same execution. Nothing is broken; the hardware is behaving exactly as specified. The explanation is that each core holds its own write in a private *store buffer* before it becomes visible to the other core, and each core is allowed to proceed to its read before its own earlier write has drained. The two reads both slip in front of the two writes. This tiny program is called the **store-buffering (SB) litmus test**, and the fact that its "impossible" outcome is in fact possible is the entire memory-consistency problem in miniature. Everything else in this paper is an elaboration of, or a response to, this one phenomenon.

### 1.3 What this paper addresses

The purpose of the full term paper is to study memory consistency models rigorously — their definitions, their realization in hardware, their cost, and how they are specified, tested, and verified. This first draft addresses the front matter of that study:

- **Problem introduction and familiarization.** We make the reader feel the problem concretely (via litmus tests) and understand its root cause (reordering plus buffering plus caching), building directly on the pipelining and memory-system material of the course.
- **The coherence/consistency distinction.** We separate two ideas that are routinely conflated, following the treatment the course texts are careful to give (Sarangi, *Next-Gen*, §9.2.3; *Basic*, §12.4.2–§12.4.3).
- **A formal problem statement.** We state precisely what a consistency model *is* as a mathematical object — a constraint on the values reads may return — so that later sections can compare models on equal footing.
- **History.** We trace how the field developed, from the pipelined-computer origins of reordering through to modern rigorous models.
- **Breadth and depth.** We survey the space of hardware models (TSO, PSO, RMO, ARM, POWER, Alpha) and language models (Java, C++11), and the two dominant specification styles (operational and axiomatic), so that the reader understands both how wide the subject is and how deep any one point in it can go.
- **Literature.** We provide an annotated survey of the primary sources, with links, that the remainder of the paper will draw on.

### 1.4 Contributions of this draft

This draft is not primarily a source of new results; it is a foundation. Its contributions are (i) a self-contained, course-anchored motivation of the consistency problem that a reader of the course can follow without external prerequisites; (ii) a consolidated history that connects the course's pipelining and multiprocessor chapters to the research literature; (iii) a comparative map of the model landscape presented as a single reference table; and (iv) an experimental plan, tied to concrete tooling, that the later sections will execute. Where the course texts stop at an introductory treatment, we mark the boundary explicitly and hand off to the primary literature.

### 1.5 Organization

Section 2 builds the background from the uniprocessor contract to the multiprocessor surprise and fixes the coherence/consistency distinction. Section 3 states the problem formally. Section 4 develops sequential consistency as the reference model and quantifies its cost. Section 5 is the history. Section 6 surveys hardware models (breadth); Section 7 covers language-level models; Section 8 covers specification and verification (depth). Section 9 is the annotated literature survey. Section 10 discusses open problems, and Section 11 lays out the plan and experimental methodology for the rest of the term paper. Section 12 concludes.

---

## 2. Background: From Uniprocessor Order to Multiprocessor Surprise

### 2.1 The uniprocessor contract

The reason a programmer can reason about a sequential program at all is a guarantee the hardware works hard to provide: **the appearance of program order**. Even though a modern pipeline fetches, decodes, and executes many instructions simultaneously, and an out-of-order core may complete them in a scrambled sequence, the architecture ensures that the *observable result* is as if the instructions had executed one at a time, in the order the program specifies. Kogge's classic treatment of pipelined computers is fundamentally about maintaining this illusion in the presence of overlap; the course's own pipelining chapter formalizes the mechanisms — data interlocks for read-after-write hazards, forwarding to avoid stalls, and precise handling of branches and exceptions (Sarangi, *Basic*, Ch. 10, esp. §10.4 hazards and §10.7 forwarding). Out-of-order machines go further and reorder execution aggressively, but the *commit* (or retirement) stage puts results back into program order, and the load-store queue enforces the necessary memory-ordering constraints so that, to a single thread, memory still behaves sequentially (Sarangi, *Next-Gen*, §4.3 on the Load-Store Queue; §4.4 on in-order commit).

The key point for our purposes: on a uniprocessor, reordering is *invisible*. The hardware is permitted to do anything internally, provided the single stream of results is indistinguishable from sequential execution. This is why a programmer of a sequential program need never learn what a store buffer is.

> **[DIAGRAM 1]** — *Caption:* "Program order vs. execution order on one core." *Description:* A small pipeline timing diagram (in the style of the course's pipeline diagrams, Sarangi *Basic* Fig. 10.x) showing instructions executing/completing out of order internally, with an arrow to a "commit in program order" box that restores the sequential appearance. Emphasize: the outside world only sees the committed order.

### 2.2 The three ingredients of reordering

To understand why multiprocessors surprise us, it helps to name the specific hardware features that reorder or delay memory operations. Each is individually benign on a uniprocessor and each is described in the course's memory-system and pipelining chapters:

1. **Out-of-order execution and dynamic scheduling.** A core issues independent instructions as their operands become ready, not in program order (Sarangi, *Next-Gen*, Ch. 2–4). Two loads to different addresses may execute in either order; a load may execute before an older store to a *different* address.
2. **Store buffers (write buffers).** To avoid stalling on the latency of making a write globally visible, a core places the write in a private FIFO buffer and continues. Subsequent reads by the same core may even read from this buffer (store-to-load forwarding) before the write is visible to others. Adve & Gharachorloo (1996) use exactly the write-buffer-with-read-bypassing optimization as their first example of how sequential consistency is violated.
3. **Caches and the interconnect.** Each core has private caches; a write propagates to other cores only through the coherence mechanism, and different cores may learn of independent writes in different orders unless the system specifically prevents it (Sarangi, *Basic*, Ch. 11 memory system; §12.4.6 coherent private caches).

The uniprocessor pipeline hides all three. In a multiprocessor, all three become externally observable through the eyes of *another* core.

> **[FIGURE 2]** — *Caption:* "Where reordering comes from." *Description:* A single core annotated with three highlighted structures — the out-of-order issue/execute logic, the store buffer, and the private cache + interconnect port — each labeled with the reordering it can introduce (e.g., store buffer — "your own later read can overtake your earlier write from another core's viewpoint"). Cross-reference to Sarangi *Next-Gen* §9.2.2.

### 2.3 Coherence is not consistency

Before going further we must fix a distinction that the course texts are careful about and that beginners routinely blur (Sarangi, *Next-Gen*, §9.2.3, "Difference between Coherence and Consistency"; *Basic*, §12.4.2–§12.4.3; H&P, 5th ed., §5.4 vs §5.6).

- **Cache coherence** concerns a *single* memory location. Informally, it guarantees that all cores agree on a single, total order of the writes to any one location, and that a read of that location eventually sees the latest write to it. Coherence is what a protocol like MSI/MESI/MOESI provides (Sarangi, *Next-Gen*, §9.4; *Basic*, §12.4.6). It is a per-address property: it says nothing about how accesses to *different* addresses are ordered relative to one another.
- **Memory consistency** concerns the ordering of accesses *across* locations. It answers questions like: if I write `X` and then write `Y`, can another core see the new `Y` but the old `X`? Coherence has nothing to say about this, because `X` and `Y` are different locations. Consistency is the system-wide rulebook that does.

A useful way to hold the distinction: **coherence makes each variable behave sanely on its own; consistency governs how variables behave with respect to each other.** A system can be perfectly coherent and still allow the store-buffering outcome of §1.2, because that outcome is about the *relative* ordering of accesses to `X` and `Y`. This is exactly why consistency is a separate, and harder, problem than coherence, and why it is the focus of this paper. The course makes the same point: coherence is necessary plumbing, but the *programmer-visible* ordering guarantees are the consistency model (Sarangi, *Next-Gen*, §9.2.2–§9.2.3).

> **[TABLE 1]** — *Caption:* "Coherence vs. Consistency at a glance." *Description:* Two-column table contrasting the two along rows: *Scope* (single location vs. across locations), *Guarantees* (single total order per location + eventual write propagation vs. permitted orderings of all memory operations), *Typical mechanism* (MSI/MESI/MOESI protocol vs. store-buffer/pipeline ordering rules + fences), *Programmer sees it as* (values are never stale forever vs. which reorderings are legal), *Course reference* (Sarangi *Next-Gen* §9.4 vs. §9.3, §9.5). This table should be placed immediately after §2.3.

### 2.4 Litmus tests: the vocabulary of the field

The store-buffering example is one of a small family of tiny two- or four-thread programs — **litmus tests** — that the field uses as its working vocabulary. Each test isolates one specific ordering question, and each memory model is characterized by which litmus outcomes it permits or forbids. We introduce the canonical ones here because they recur throughout the paper; the experimental part of the term paper (Section 11) will run them on real hardware.

**Store Buffering (SB).** As above. Initially `X = Y = 0`.

```
Core 1:            Core 2:
  X = 1              Y = 1
  r1 = Y            r2 = X
```

Question: can `r1 = 0 ∧ r2 = 0`? Under sequential consistency, **no**. Under TSO (x86), **yes** — this is the defining relaxation of TSO, caused by store buffers. (Adve & Gharachorloo 1996; Sewell et al. 2010.)

**Message Passing (MP).** Initially `data = 0`, `flag = 0`.

```
Core 1 (producer):   Core 2 (consumer):
  data = 42            while (flag == 0) {}
  flag = 1             r = data
```

Question: when Core 2 sees `flag == 1`, is it guaranteed that `r == 42`? On a strongly ordered model (SC, and — for this particular pattern — TSO, which keeps store→store order), yes. On weakly ordered models (ARM, POWER, Alpha), **not without a barrier**: the two writes on Core 1 can become visible out of order, or the two reads on Core 2 can be reordered, so the consumer can see the flag set but read stale data. This is the single most important pattern in real concurrent code, and the reason release/acquire fences exist. (Sarangi, *Next-Gen*, §9.5; Maranget, Sarkar & Sewell 2012.)

**Load Buffering (LB).** Initially `X = Y = 0`.

```
Core 1:            Core 2:
  r1 = X             r2 = Y
  Y = 1              X = 1
```

Question: can `r1 = 1 ∧ r2 = 1`? Requires each core's read to be reordered after its write. Forbidden by SC and TSO; allowed by some weak models. Distinguishes models that keep read→write order from those that do not.

**Independent Reads of Independent Writes (IRIW).** Initially `X = Y = 0`. Two cores write; two other cores read both variables in opposite orders. The question is whether two observer cores can disagree on the order in which the two independent writes occurred. Models that permit this are **not multiple-copy atomic** (a write can become visible to some cores before others); classic POWER and older ARM permit it, while x86-TSO does not. This test is the cleanest probe of *write atomicity*, one of the deepest axes in the whole subject. (Adve & Gharachorloo 1996; Alglave, Maranget & Tautschnig 2014.)

> **[TABLE 2]** — *Caption:* "Canonical litmus tests and which models permit the 'surprising' outcome." *Description:* Rows = SB, MP (without barriers), LB, IRIW, plus Dekker-style and coherence sanity tests; columns = SC, x86-TSO, SPARC PSO, SPARC RMO, ARMv7/POWER, ARMv8 (multicopy-atomic). Each cell: Forbidden / Allowed. This is a keystone reference table; place it here and refer back to it in Sections 4 and 6.

> **[DIAGRAM 2]** — *Caption:* "Store buffering, mechanistically." *Description:* Two cores each with a store buffer in front of a shared memory. Show Core 1's `X=1` sitting in its buffer while its `r1=Y` reads memory (still 0), and symmetrically for Core 2, producing `r1=r2=0`. This is the operational (abstract-machine) explanation that Sewell et al. (2010) make precise for x86-TSO.

### 2.5 Why the reader should care: three stakes

The problem is not academic. The consistency model matters for three distinct reasons, which recur throughout the literature (Adve & Gharachorloo 1996 frame them as the canonical trio):

1. **Correctness.** Synchronization code — locks, lock-free data structures, the internals of language runtimes and OS kernels — is correct only relative to a memory model. The same C code can be correct on x86 and subtly broken on ARM. Sewell et al. (2010) demonstrate this by verifying a Linux spinlock against x86-TSO; the proof depends on TSO's specific guarantees.
2. **Portability.** Because models differ, a program reasoned about on one architecture may misbehave on another. This is what pushed the problem *up* into programming languages: Java and C++ define their own memory models so that portable concurrent code has a fixed target regardless of the underlying hardware (Manson, Pugh & Adve 2005; Boehm & Adve 2008).
3. **Performance.** Every ordering guarantee has a cost. Enforcing sequential consistency naïvely means a core often cannot proceed past a memory operation until it is globally visible, throwing away the very buffering and reordering that make cores fast. Relaxed models exist precisely to recover that performance; the design tension of the entire field is *how much ordering to give up for how much speed, and how to give programmers back just enough control (via fences) to be correct.* Quantifying this trade-off is one of the experimental goals of the term paper (Section 11).

With the problem felt and its vocabulary in place, we can now state it precisely.

---

## 3. The Problem Statement

### 3.1 The objects we reason about

To state the problem precisely we need a small amount of vocabulary, standard across the literature (Lamport 1979; Adve & Gharachorloo 1996; Sarangi, *Next-Gen*, §9.3.1 "Sequential and Parallel Executions"; Alglave, Maranget & Tautschnig 2014).

- A **memory operation** is a read (load) or a write (store) of a value to a shared memory location. We write `W(X, v)` for "write value `v` to location `X`" and `R(X): v` for "read of `X` returning `v`."
- Each core executes a sequence of operations in **program order** — the order given by its own instruction stream. Program order is a *per-core* total order.
- An **execution** of a multithreaded program is the collection of operations actually performed by all cores, together with, for each read, the value it returned. Two runs of the same program can be different executions.
- Various **orders** are defined over the operations of an execution: program order (per core), a *coherence order* (the total order of writes to each single location), a *reads-from* relation (which write each read observed), and — for models defined that way — a global *memory order*.

The consistency model's job is to say **which executions are legal**: given the program, which combinations of read-return-values can actually occur on a conforming machine.

### 3.2 The consistency question, stated

> **Problem (Memory Consistency).** Given a shared-memory multiprocessor and a multithreaded program, characterize exactly the set of executions the machine may produce — equivalently, characterize, for every read in every possible execution, the set of values that read is permitted to return.

A **memory consistency model** is a specification that answers this question. Formally it is a predicate on executions: it partitions all conceivable executions of a program into *allowed* and *forbidden*. Adve & Gharachorloo (1996) phrase it as: the consistency model "places restrictions on the values that can be returned by a read in a shared-memory program execution." On a uniprocessor the answer is trivial — a read returns the value of the most recent write to that location in program order — because "most recent" is well defined by the single program order. In a multiprocessor there is no single program order across cores, and "most recent" becomes ambiguous. The model is exactly what disambiguates it.

Two things make this a genuine specification problem rather than a mere implementation detail:

- **It is a contract, not a description of one machine.** The model is what hardware designers promise and what software may rely on. Two very different microarchitectures can implement the same model; one microarchitecture revision must not break software by silently changing the model. This is why vendors publish models, and why the research program of the 2000s–2010s was to make those published models *rigorous* rather than prose folklore (Sewell et al. 2010).
- **It is where correctness, portability, and performance are jointly negotiated.** A stronger model (fewer allowed executions) is easier to program against but harder to implement quickly; a weaker model is the reverse. The whole design space lives between these poles.

### 3.3 Two ways to give an answer

Historically, models are specified in one of two equivalent-in-spirit styles, both of which the term paper will use (developed further in Section 8):

- **Operational / abstract-machine style.** Define an idealized machine — e.g., cores plus explicit store buffers plus a shared memory, with rules for when a buffered write drains — and declare an execution legal iff that abstract machine can produce it. This is intuitive and close to hardware; x86-TSO is presented this way (Sewell et al. 2010), and the course's implementation section is in this spirit (Sarangi, *Basic*, §12.4.7 "Implementing a Memory Consistency Model").
- **Axiomatic style.** Define mathematical relations over the operations (program order, reads-from, coherence, derived "happens-before" style orders) and declare an execution legal iff those relations satisfy stated acyclicity/consistency axioms. This is compact, comparable across models, and tool-friendly; it underlies the `herd` tool and the "herding cats" framework (Alglave, Maranget & Tautschnig 2014).

The two styles are complementary: operational models build intuition and match implementations; axiomatic models enable comparison and automated testing. A recurring theme of the field — and a point we will return to in Section 8 — is *proving that an operational and an axiomatic model of the "same" architecture actually coincide*.

> **[FIGURE 3]** — *Caption:* "A consistency model as a filter on executions." *Description:* A funnel diagram. Left: the space of "all conceivable executions" of a program (all assignments of return values to reads). The consistency model is drawn as a filter that passes only the *allowed* executions to the right. Show SC as a narrow filter (few executions pass) and a relaxed model as a wider filter (more pass, including the "surprising" SB outcome). This visualizes §3.2 and sets up §4.

### 3.4 Scope note

The full problem has several axes we will unfold gradually: (a) which *program-order pairs* a model may reorder (W→R, W→W, R→R, R→W); (b) whether a core may **read its own write early** (store-to-load forwarding); (c) whether a core may **read another core's write early** (write atomicity / multi-copy atomicity); and (d) what **fences/barriers** the model provides to restore order on demand. Sections 4–6 populate these axes; Table 3 (in Section 6) collects them into a single comparison. This decomposition follows Adve & Gharachorloo (1996) and is mirrored in the course's treatment of "popular memory models" (Sarangi, *Next-Gen*, §9.5).

---

## 4. Sequential Consistency: The Reference Model

### 4.1 Lamport's definition

Every discussion of consistency starts from the model that formalizes the naïve intuition, and it has a precise origin. In 1979, Leslie Lamport gave the definition in a two-page paper (Lamport 1979):

> A multiprocessor is **sequentially consistent** if the result of any execution is the same as if the operations of all the processors were executed in some single sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.

Two conditions are packed into this sentence. First, there must exist *some* single total order (an interleaving) of all memory operations from all cores. Second, that interleaving must respect each core's program order. A read returns the value written by the most recent write to the same location in this global interleaving. SC is thus the model in which the multiprocessor behaves like a set of threads time-sliced onto one processor with a single memory — the mental model most programmers unconsciously assume (Sarangi, *Next-Gen*, §9.3.2; H&P, 5th ed., §5.6).

Under SC, the store-buffering outcome `r1 = r2 = 0` is **impossible**: any interleaving that respects both program orders must place at least one write before the other core's read. Working through the four operations confirms there is no legal interleaving yielding both zeros. This is the sense in which SC matches intuition — and, as we will see, the sense in which real hardware departs from it.

> **[DIAGRAM 3]** — *Caption:* "Sequential consistency as interleaving." *Description:* The classic picture (as in Lamport 1979 and reproduced in Sarangi *Next-Gen* §9.3.2 / Adve–Gharachorloo Fig. 2): each core is a vertical list of its operations; a single "switch" selects operations one at a time into a global sequence that respects each list's internal order, feeding one shared memory. Show that SB's double-zero outcome corresponds to no valid switch schedule.

### 4.2 Sufficient conditions and why SC is expensive

Lamport's paper did more than define SC; it gave *sufficient conditions* for a machine to be sequentially consistent — essentially, that each processor issues its memory operations one at a time in program order and that memory services requests respecting per-location order. The cost is visible immediately: to guarantee program order between, say, a write and a following read, a naïve SC implementation must not let the read execute until the write is globally performed. That directly forbids the store buffer (which is why SB is impossible under SC), and store buffers are one of the cheapest and most effective latency-hiding tricks a core has. Enforcing SC therefore tends to serialize memory operations and expose their full latency.

The course texts make this cost concrete in their implementation discussion (Sarangi, *Basic*, §12.4.7; *Next-Gen*, §9.3.5 "SC using Synchronisation Instructions"), and it is the reason essentially no high-performance processor implements SC directly. A great deal of clever microarchitecture (speculative execution of loads with rollback on detected violations, as in Gharachorloo, Gupta & Hennessy's techniques, and MIPS R10000-style load-queue snooping) has been devised to *recover* SC's guarantees while keeping some of the performance — an active area to this day. But the dominant industrial response was different: **relax the model**.

### 4.3 The theory underneath: coherence as per-location SC

A conceptual bridge worth stating: coherence can be viewed as "sequential consistency, one location at a time." Coherence requires a single total order of the writes to each individual location that all cores respect; SC requires a single total order over *all* operations to *all* locations. The course develops this relationship carefully (Sarangi, *Next-Gen*, §9.3.4 "From PLSC to Coherence"), which is a good example of how the introductory framing (coherence vs consistency, §2.3) is made rigorous. We flag this as a place where the term paper's later, deeper sections will pick up the formal thread; for the introduction it suffices to see coherence and consistency as the same kind of constraint applied at two different scopes.

> **[TABLE 3-preview]** — *Caption:* "SC as the top of the ordering hierarchy." *Description:* A single-row-highlighted version of the main model table (Table 3, Section 6) with SC at the top ("preserves all four program-order pairs; writes atomic") and an arrow downward labeled "relax to gain performance." Serves as a visual hinge between Section 4 and Section 6.

---

## 5. A History of Memory Consistency Models

The field did not arrive fully formed. It grew in response to hardware that kept getting faster in ways that broke the simple picture, and to software that kept needing a stable target. This section traces that development. It doubles as a map of the primary literature the term paper relies on.

> **[FIGURE 4]** — *Caption:* "Timeline of memory consistency models, 1970s–2020s." *Description:* A horizontal timeline with milestones: (1970s) pipelining and out-of-order completion (Kogge); 1979 Lamport defines SC; 1986 Dubois–Scheurich–Briggs weak ordering / access buffering; 1990 Adve–Hill "weak ordering — a new definition," Gharachorloo et al. release consistency; 1990s SPARC TSO/PSO/RMO, Alpha, PowerPC; 1996 Adve–Gharachorloo tutorial; 2005 Java Memory Model (JSR-133); 2008 C++ memory model foundations; 2010 x86-TSO; 2011–2012 POWER/ARM operational + axiomatic models; 2014 "herding cats"; 2018 ARMv8 becomes multicopy-atomic; 2020 Primer 2nd ed. Use this as the anchor figure for Section 5.

### 5.1 Prehistory: reordering is born on the uniprocessor (1960s–1970s)

The seeds of the problem predate multiprocessors. As soon as pipelined and overlapped machines executed operations in an order other than program order — the subject of Kogge's *The Architecture of Pipelined Computers* — the gap between "the order written" and "the order performed" existed. On a uniprocessor that gap is invisible, but the *machinery* that creates it (buffering, overlap, out-of-order completion) is exactly what later becomes visible in a multiprocessor. Lamport's own framing in the 1979 paper is telling: he notes that "many large sequential computers execute operations in a different order than is specified by the program," and that this is harmless for one processor but not for many. In this sense the consistency problem is the multiprocessor shadow of uniprocessor pipelining — which is precisely why the course sequences pipelining (Sarangi *Basic* Ch. 10, *Next-Gen* Ch. 2–4) before multiprocessor consistency (Ch. 12 / Ch. 9).

### 5.2 1979: Lamport names the target

Lamport's two-page note (Lamport 1979) is the field's founding document. By giving a precise definition of correct multiprocessor execution — sequential consistency — it converted an intuition into a criterion that hardware could be measured against and, later, deliberately relaxed away from. Its influence is hard to overstate: SC remains the yardstick against which every other model is described ("this model relaxes SC by allowing …"). *Link:* Lamport's own copy at his publications page; DOI 10.1109/TC.1979.1675439 (see references).

### 5.3 The 1980s: relaxation begins

Through the 1980s, architects realized that SC left too much performance on the table and began defining weaker models that formalized specific relaxations:

- **Processor consistency / total-store-order lineage.** Allowing a core to buffer its writes (and read its own buffered writes early) while keeping writes in order gives what later crystallized as **Total Store Order (TSO)**. Goodman's *processor consistency* and the SPARC architecture's TSO are of this lineage.
- **Weak ordering.** Dubois, Scheurich & Briggs (1986) introduced buffering-based relaxations and the idea that ordering need only be enforced at *synchronization* points, not between ordinary accesses. Adve & Hill (1990), "Weak Ordering — A New Definition," reframed weak ordering in *programmer-centric* terms: if a program is properly synchronized (data-race-free), the hardware can reorder ordinary accesses freely and still appear sequentially consistent to that program. This "SC-for-DRF" idea (Section 7.1) became the philosophical backbone of later *language* models.

The through-line of the decade: give up ordering between ordinary accesses, and give programmers explicit synchronization operations (fences) to reimpose order where they actually need it.

### 5.4 1990: release consistency and the tutorial era

Gharachorloo, Lenoski, Laudon, Gibbons, Gupta & Hennessy (1990), "Memory Consistency and Event Ordering in Scalable Shared-Memory Multiprocessors," introduced **release consistency (RC)**, which refines weak ordering by distinguishing *acquire* operations (which must complete before following accesses) from *release* operations (which must wait for preceding accesses). This acquire/release distinction is not a historical curiosity: it is *exactly* the vocabulary that C++11 later adopted for its atomics (`memory_order_acquire`, `memory_order_release`; Section 7.3), and it is the model realized in the Stanford DASH multiprocessor. Release consistency is the point where the hardware-model literature and the eventual language-model literature visibly converge.

By the mid-1990s the proliferation of incompatible models had itself become a problem, and Adve & Gharachorloo's 1996 tutorial (first circulated as WRL Research Report 95/7, 1995) organized the zoo into a coherent framework built on two questions — *which program-order pairs may be reordered?* and *may a write be read early (by the writer, or by others)?* This tutorial is still the single best on-ramp to the subject and is a primary reference for this paper. *Link:* WRL RR 95/7 / IEEE Computer 29(12):66–76, 1996 (see references).

### 5.5 The 2000s: the problem moves into programming languages

As multicore went mainstream, the portability stake (§2.5) became acute: application and library programmers write in high-level languages and cannot target a different hardware model per platform. The response was to define **language-level memory models**.

- **Java (2005).** Manson, Pugh & Adve (2005), the Java Memory Model (JSR-133), was the first serious attempt to give a mainstream language a rigorous concurrency semantics — guaranteeing SC for data-race-free programs while still bounding the behavior of racy programs enough to preserve safety and security properties. It proved surprisingly subtle, exposing how hard "give a portable model that both admits compiler optimizations and forbids nonsense" really is.
- **C++ (2008 — C++11).** Boehm & Adve (2008), "Foundations of the C++ Concurrency Memory Model," laid the groundwork for the model standardized in C++11, which exposes a spectrum of `std::atomic` operations tagged with explicit `memory_order` values (relaxed, acquire, release, acq_rel, seq_cst). C++ made the hardware/language correspondence explicit and gave programmers direct, portable control over the ordering/performance trade-off. This is the model the term paper's microbenchmarks (Section 11) will exercise directly.

### 5.6 The 2010s: rigorous, tool-backed hardware models

The final act (so far) was to make *hardware* models rigorous. For decades, vendor prose specifications were ambiguous and sometimes unsound. Two Cambridge-led lines of work fixed this:

- **x86-TSO (2010).** Sewell, Sarkar, Owens, Zappa Nardelli & Myreen (2010) reviewed the Intel/AMD prose specs, showed they were ambiguous or too weak, and gave a single rigorous model — x86-TSO — in two proven-equivalent styles: an operational abstract machine with explicit store buffers, and an axiomatic model. Crucially, it is *usable*: they reason about a Linux spinlock with it. x86-TSO is now the de facto reference for x86 concurrency. *Link:* CACM 53(7):89–97, 2010; DOI 10.1145/1785414.1785443; project page cl.cam.ac.uk/~pes20/weakmemory.
- **POWER and ARM (2011–2012).** Sarkar, Sewell, Alglave, Maranget and colleagues produced operational and axiomatic models for the substantially weaker POWER and ARM architectures, including the accessible tutorial by Maranget, Sarkar & Sewell (2012). These architectures permit reorderings (and, historically, non-multicopy-atomic writes — recall IRIW, §2.4) that x86 does not, making them far harder to model.
- **"Herding cats" (2014).** Alglave, Maranget & Tautschnig (2014) unified the axiomatic approach into a single framework and the `herd` simulator, enabling models to be written as sets of axioms, simulated, and tested by *data-mining* real hardware with millions of litmus runs. This is the methodological basis for the litmus-testing part of the term paper. *Link:* ACM TOPLAS 36(2):7, 2014; DOI 10.1145/2627752.
- **ARMv8 becomes multicopy-atomic (2018).** In a notable case of models informing design, ARM revised its architecture to be *other-multicopy-atomic*, ruling out some of the hardest IRIW-style behaviors and simplifying reasoning — evidence that the rigorous-modeling program fed back into industrial architecture.

The 2020 second edition of the *Primer on Memory Consistency and Cache Coherence* (Nagarajan, Sorin, Hill & Wood 2020) — now open access — consolidates this era, adding chapters on accelerator/GPU consistency and on formal tools, and is the standard modern reference.

> **[GRAPH 1]** — *Caption:* "Strength vs. permissiveness of major hardware models." *Description:* A one-dimensional axis from 'strongest / most ordered' (SC) to 'weakest / most relaxed' (Alpha, classic POWER), placing TSO (x86, SPARC-TSO), PSO, RMO, ARMv7, ARMv8, POWER along it, annotated with the key relaxation each one adds relative to its left neighbor. This graphically summarizes Section 5.6 and previews Table 3.

### 5.7 Reading the history as a single argument

Stepping back, the history has a clean logical shape. (1) Uniprocessor speed tricks create reordering, invisibly. (2) Multiprocessors make that reordering visible; Lamport defines the ideal (SC) it violates. (3) SC is too slow, so the 1980s–90s invent relaxed hardware models and give programmers fences. (4) Too many incompatible models make software non-portable, so the 2000s lift the problem into languages (Java, C++). (5) Vendor specs are too vague to build on, so the 2010s make hardware models rigorous and tool-backed, and the tools even feed back into hardware design. The term paper's remaining sections live at stage (5) and its interface with stages (3)–(4).

---

## 6. The Landscape of Hardware Memory Models (Breadth)

This section maps the breadth of the subject: the major hardware models and the axes along which they differ. The organizing insight, due to Adve & Gharachorloo (1996) and mirrored in the course (Sarangi, *Next-Gen*, §9.5 "Memory Models," esp. §9.5.7 "Popular Memory Models"), is that almost every hardware model can be described by answering a few questions:

- **Which program-order pairs may be reordered?** There are four: write→read (W→R), write→write (W→W), read→read (R→R), and read→write (R→W). A model is characterized by which of these it *preserves* and which it *relaxes*.
- **May a core read its own write early** (before the write is globally visible), via store-to-load forwarding from its buffer?
- **May a core read another core's write early** — i.e., is the model *multi-copy atomic* (all cores see writes in a consistent global order) or not?
- **What fences does it provide** to reimpose order where the program needs it?

### 6.1 Sequential Consistency (SC) — the ceiling

Preserves all four program-order pairs; writes are atomic. No surprising litmus outcomes. Easiest to reason about, hardest to implement fast. Realized directly by essentially no mainstream high-performance CPU, though the MIPS R10000 famously provided an SC-like guarantee via speculation. (Section 4.)

### 6.2 Total Store Order (TSO) — x86 and SPARC-TSO

TSO relaxes **only W→R**: a core may let a later read overtake an earlier write to a *different* location, because the write sits in a FIFO store buffer. It keeps W→W, R→R, R→W. It permits a core to **read its own write early** (store-to-load forwarding) but is otherwise **multi-copy atomic**: all cores agree on a single order of writes, so IRIW is forbidden. TSO is the model of the x86 architecture (as rigorously pinned down by Sewell et al. 2010) and of SPARC's TSO mode. Consequences:

- **SB is allowed** (the W→R relaxation is exactly the store-buffer effect).
- **MP works without barriers** (W→W is preserved, so the producer's `data` write reaches memory before its `flag` write; R→R is preserved, so the consumer reads in order).
- **IRIW is forbidden** (writes are atomic).
- **Dekker's and Peterson's mutual-exclusion algorithms need an explicit fence** (`MFENCE` on x86) between the store and the subsequent load, precisely because of the W→R relaxation.

TSO is the "friendliest relaxed model": strong enough that a lot of naïve code accidentally works, weak enough to keep store buffers. This is why x86 concurrency bugs often hide until code is ported to a weaker platform.

> **[DIAGRAM 4]** — *Caption:* "The x86-TSO abstract machine." *Description:* Redraw Sewell et al. (2010)'s block diagram: N cores, each with a FIFO store buffer, all attached to a single shared memory behind a global lock; rules: reads check own store buffer first then memory; a fence drains the buffer. Label how this machine produces SB but forbids IRIW.

### 6.3 Partial Store Order (PSO) and Relaxed Memory Order (RMO) — SPARC's weaker modes

SPARC defined a ladder of models. **PSO** additionally relaxes **W→W**: two writes by the same core may become visible out of order, so even MP can break without a store→store barrier. **RMO** relaxes read-ordering as well (R→R and R→W), approaching a fully weak model. The SPARC ladder (TSO → PSO → RMO) is the cleanest textbook illustration that "relaxed" is not one model but a spectrum, and the course uses it in exactly this pedagogical role (Sarangi, *Next-Gen*, §9.5.7).

### 6.4 Weakly ordered models — ARM and IBM POWER

ARM (v7 and earlier v8) and POWER are **weakly ordered**: by default they relax *all four* program-order pairs for accesses to different locations, keeping only the per-location coherence order and true (data/address/control) dependencies. Ordinary code must use explicit barriers (`DMB`/`DSB`/`ISB` on ARM; `sync`/`lwsync`/`isync`/`eieio` on POWER) to enforce ordering. Historically, POWER and ARMv7 were also **not multi-copy atomic** — a write could be seen by different cores at different times — so **IRIW is allowed**, which is the deepest departure from intuition and the hardest thing to model. Maranget, Sarkar & Sewell (2012) is the standard tutorial; Sarkar et al. and Alglave et al. give the full models. As noted (§5.6), **ARMv8 was later revised to be other-multicopy-atomic (2018)**, ruling out the worst IRIW behaviors — a rare case of formal modeling changing an ISA.

### 6.5 Alpha — a cautionary tale

DEC Alpha was so aggressively relaxed that it did not even guarantee ordering between a load of a pointer and a dependent load through that pointer *without* a memory barrier — famously requiring a `rmb` where every other architecture relied on the data dependency. This caused real, subtle bugs (the Linux kernel carried `read_barrier_depends()` largely for Alpha) and is the standard example that "you can relax too far." It is worth including as the extreme point of the spectrum.

### 6.6 Fences/barriers — the escape hatch

Every relaxed model supplies **fence** (barrier) instructions that reimpose ordering on demand. A fence between two operations forbids the model from reordering across it. Fences are how programmers buy back exactly the ordering they need and no more — the mechanism that makes relaxed models usable. Their cost is real (a fence can stall a core for tens to hundreds of cycles by draining buffers or waiting for global visibility), which is why over-fencing is a performance bug and under-fencing is a correctness bug. Quantifying fence cost across `memory_order` levels is one of the microbenchmark goals of the term paper (Section 11). The course covers barrier-based enforcement under "SC using synchronisation instructions" (Sarangi, *Next-Gen*, §9.3.5) and in the implementation section (Sarangi, *Basic*, §12.4.7).

> **[TABLE 3]** — *Caption:* "Comparison of major hardware memory models." *Description:* The keystone comparison table. Rows = SC, TSO (x86/SPARC-TSO), PSO, RMO, ARMv7/POWER (weak), ARMv8 (weak but multicopy-atomic), Alpha. Columns = W→R relaxed?, W→W relaxed?, R→R relaxed?, R→W relaxed?, reads own write early?, multi-copy atomic (writes atomic)?, representative fences, example real ISA. Fill each cell Yes/No. Add a footnote row mapping each model to the litmus tests it fails from Table 2. This table is the single most important reference object in the paper; it should be full-width and may span landscape orientation in LaTeX.

### 6.7 GPUs and heterogeneous systems (breadth beyond CPUs)

The breadth of the subject now extends past CPUs. GPUs and heterogeneous CPU-GPU systems have their own, often *scoped*, consistency models (ordering guarantees that differ within a thread block vs. across the whole device), and getting these right is an active research area — the 2020 *Primer* added an entire chapter on non-CPU accelerators for this reason (Nagarajan et al. 2020). We flag this as a breadth frontier the term paper can point to, though the experimental work will stay CPU-focused.

---

## 7. Language-Level Memory Models

Hardware models are not the whole story. Application programmers write in C, C++, Java, Rust, Go — not in assembly — and the compiler is *itself* a source of reordering (it hoists, sinks, and eliminates memory accesses). A correct concurrent program therefore needs a contract with the *language*, not just the hardware. This section covers that layer, which the term paper's C++ microbenchmarks (Section 11) sit squarely on top of.

### 7.1 The unifying idea: SC for data-race-free programs (SC-DRF)

The single most important idea in language memory models is **SC-DRF**, whose lineage runs from Adve & Hill (1990) through the Java and C++ models: *if a program has no data races — every pair of conflicting accesses is ordered by synchronization — then the programmer is guaranteed sequential consistency, even though the underlying hardware and compiler are relaxed.* The bargain is: programmers promise to synchronize properly (use locks or atomics for shared mutable data); in return, the system promises the intuitive SC semantics. Relaxation is confined to (a) ordinary accesses in correctly-synchronized code, where it is invisible, and (b) explicitly-tagged low-level atomics, where the programmer opts into weaker guarantees knowingly. The course connects to this through its data-race material (Sarangi, *Next-Gen*, §9.6, esp. §9.6.3 "Properly Synchronised Programs" and §9.6.4 "DRF Memory Models").

> **[FIGURE 5]** — *Caption:* "The SC-DRF bargain." *Description:* A flowchart: "Is the program data-race-free?" — Yes — "Guaranteed to behave with sequential consistency"; No — "Behavior governed by weak/atomics rules (possibly undefined for racy access in C++)." Annotate with where locks and `std::atomic` fit. This frames Section 7.

### 7.2 The Java Memory Model (2005)

Java, being memory-safe and sandboxed, cannot allow racy programs to do *anything* — undefined behavior would break the security model. So the Java Memory Model (Manson, Pugh & Adve 2005) had the hard job of giving SC-DRF *and* bounding the behavior of racy programs (via the "happens-before" relation and causality constraints that forbid values appearing "out of thin air"). It is the first industrial-strength language memory model, and its difficulty — several proposed versions had subtle flaws — is itself an important lesson about how non-obvious this territory is.

### 7.3 The C++ Memory Model and `std::atomic` (C++11)

C++, prioritizing performance and control, exposes the ordering/performance trade-off *directly* to the programmer through `std::atomic` operations parameterized by a `memory_order` (Boehm & Adve 2008 laid the foundations; standardized in C++11):

- `memory_order_seq_cst` — the default; participates in a single total order of all seq_cst operations (SC-like, strongest, most expensive).
- `memory_order_acquire` / `memory_order_release` — the acquire/release pair from *release consistency* (§5.4): a release-store synchronizes-with an acquire-load of the same variable, giving exactly the ordering MP needs, without a full fence.
- `memory_order_acq_rel` — both, for read-modify-write operations.
- `memory_order_relaxed` — atomicity only, no ordering; the cheapest, used for counters and flags where ordering is not required.

This is the cleanest place to *measure* the trade-off empirically, because the same program can be recompiled with different `memory_order` tags and its performance and correctness observed. That is precisely the design of the term paper's microbenchmark suite (Section 11), where we will compile the same producer/consumer and counter workloads under each `memory_order` and measure with hardware counters.

> **[TABLE 4]** — *Caption:* "C++11 `memory_order` levels: guarantee vs. cost, and hardware mapping." *Description:* Rows = seq_cst, acq_rel, acquire, release, relaxed. Columns = ordering guaranteed, typical use, mapping on x86-TSO (often free for acquire/release loads/stores; fence for seq_cst), mapping on ARM/POWER (barriers required), expected relative cost. This table directly frames the microbenchmark experiments and connects Section 7 to Section 6.

### 7.4 The compiler as a reordering agent

A subtle but crucial point the term paper will emphasize: even on a strongly ordered CPU like x86, a program can exhibit weak-memory bugs because the *compiler* reordered accesses at `-O2`. The language memory model is what makes the compiler's optimizations legal only up to the point where they remain invisible to correctly-synchronized code. Vafeiadis et al. (2015) showed that some "obvious" compiler optimizations are actually unsound under the C11 model — evidence that the language layer is as subtle as the hardware layer. This motivates measuring *compiled* behavior (Section 11), not just reasoning about source.

---

## 8. Specifying and Verifying Memory Models (Depth)

Where Sections 6–7 showed breadth, this section shows depth: how much rigor and machinery a single point in the space demands. This is where an introductory course treatment (which necessarily stops at definitions and examples) hands off to the research literature, and it is where much of the term paper's technical contribution will live.

### 8.1 Two specification styles, and proving they agree

As introduced in §3.3, models are given operationally (abstract machine) or axiomatically (relations + acyclicity axioms). A central technical activity is *proving the two coincide* for a given architecture — e.g., that the x86-TSO store-buffer machine accepts exactly the executions the x86-TSO axioms allow (Sewell et al. 2010, done in the HOL4 proof assistant). This matters because the operational form guides implementers and the axiomatic form guides tool-builders and programmers; a proof that they agree means both communities are talking about the same model.

> **[DIAGRAM 5]** — *Caption:* "Operational vs. axiomatic, and the equivalence obligation." *Description:* Left box: abstract machine (store buffers, drain rules) generating a set of executions. Right box: axioms over program-order/reads-from/coherence relations accepting a set of executions. A double-headed arrow labeled "equivalence theorem (e.g., proved in HOL4 for x86-TSO)" between them. Cross-reference Sewell et al. 2010 and Alglave et al. 2014.

### 8.2 Litmus testing and the `herd`/`litmus` toolchain

The empirical backbone of the field is **litmus testing**: take a small program with a known "interesting" outcome, run it billions of times on real hardware to see whether the outcome occurs, and compare against what a candidate model predicts. The `herdtools7` suite (the `litmus7` tool to run tests on hardware, and the `herd7` tool to simulate a model) operationalizes this and is the direct descendant of the "herding cats" methodology (Alglave, Maranget & Tautschnig 2014). This is exactly the tooling the term paper will use for its hardware-observation experiments (Section 11): we will run canonical tests (SB, MP, LB, IRIW) on an x86 machine, confirm which outcomes are observable, and cross-check against the `herd7` prediction under an x86-TSO `.cat` model.

> **[FIGURE 6]** — *Caption:* "The litmus-testing loop." *Description:* A cycle: (1) write litmus test — (2a) run on real hardware with `litmus7`, collecting outcome histograms; (2b) simulate with `herd7` under a `.cat` model; (3) compare hardware-observed outcomes vs. model-permitted outcomes; (4) refine model or explain hardware. This figure sets up the experimental methodology.

### 8.3 The `.cat` language and axiomatic models as executable artifacts

A significant modern development is that axiomatic models are written in a small domain-specific language (`.cat`) as sets of relation definitions and acyclicity constraints, and then *executed* by `herd7`. A memory model becomes a short, precise, runnable file — the x86-TSO, ARMv8, POWER, RISC-V, and C++11 models all exist as `.cat` files. This closes the loop between specification and testing and is the reason the field is now unusually rigorous for computer architecture. The term paper will present and (lightly) modify a `.cat` model as part of its depth contribution.

### 8.4 Verifying hardware and compilers against models

Beyond testing, there is *verification*: proving that a microarchitecture or a compiler respects a model. This includes microarchitectural consistency verification (checking an RTL/pipeline design against its ISA model), compiler-correctness results (that a compiler's mapping from C++11 atomics to ARM/POWER barriers is sound — e.g., the "trailing-sync"/"leading-sync" mappings proved for POWER), and ongoing work on RISC-V's RVWMO model and GPU models. A 2025 survey (Sha & Wang / "Weak Memory Model Formalisms: Introduction and Survey," arXiv:2508.04115) collects the current state; we cite it as an entry point for the term paper's deeper later sections. Recent verification work continues actively (e.g., model-checking approaches to consistency, 2026), indicating the area is far from closed.

> **[TABLE 5]** — *Caption:* "Specification and verification approaches." *Description:* Rows = operational abstract machine, axiomatic (.cat), litmus testing on hardware, model equivalence proofs, microarchitectural verification, compiler-mapping correctness. Columns = what it produces, example tool/artifact, example citation, where the term paper uses it. Serves as the roadmap for the paper's technical (depth) sections.

### 8.5 Where this draft hands off

The introduction stops here: it has established that models can be specified two ways, tested against hardware with `herdtools7`, and verified with proof assistants and model checkers. The subsequent sections of the term paper will (i) present the x86-TSO `.cat` model in full and run the litmus suite against real hardware; (ii) build the C++11 microbenchmarks to measure the ordering/performance trade-off; and (iii), budget permitting, use gem5 to observe how microarchitectural parameters (store-buffer depth, speculation) affect observable orderings. Section 11 details this plan.

---

## 9. Annotated Literature Survey

This section is an annotated map of the primary sources, grouped by role, so the later sections of the term paper can cite from a curated base. Full bibliographic details and links are in the References. Annotations state *why* each source matters and *where* the paper uses it.

**Foundational definitions**

- *Lamport (1979), "How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs."* The definition of sequential consistency. Two pages; the origin point. Used in Section 4 as the reference model.
- *Adve & Gharachorloo (1996), "Shared Memory Consistency Models: A Tutorial."* The best synthesis of the relaxed-model zoo, organized by the program-order-relaxation and write-atomicity axes. Used as the backbone of Sections 3.4, 6.

**Relaxed-model origins**

- *Dubois, Scheurich & Briggs (1986), "Memory Access Buffering in Multiprocessors."* Early formal relaxation via access buffering; introduced the idea of ordering only at synchronization. Section 5.3.
- *Adve & Hill (1990), "Weak Ordering — A New Definition."* The programmer-centric reframing (SC-for-DRF) that later underlies language models. Sections 5.3, 7.1.
- *Gharachorloo et al. (1990), "Memory Consistency and Event Ordering in Scalable Shared-Memory Multiprocessors."* Release consistency and the acquire/release distinction; realized in Stanford DASH; vocabulary reused by C++11. Sections 5.4, 7.3.

**Language-level models**

- *Manson, Pugh & Adve (2005), "The Java Memory Model."* First industrial language memory model; SC-DRF plus causality bounds for racy programs. Section 7.2.
- *Boehm & Adve (2008), "Foundations of the C++ Concurrency Memory Model."* Basis of C++11 atomics/`memory_order`. Section 7.3; foundation for the microbenchmarks.
- *Vafeiadis et al. (2015), "Common Compiler Optimisations Are Invalid in the C11 Memory Model and What We Can Do About It."* Shows the compiler layer is as subtle as hardware. Section 7.4.

**Rigorous hardware models and tooling**

- *Sewell, Sarkar, Owens, Zappa Nardelli & Myreen (2010), "x86-TSO."* The rigorous x86 model in operational + axiomatic form, proved equivalent in HOL4, and shown usable on a real spinlock. Sections 6.2, 8.1; reference model for the hardware experiments.
- *Maranget, Sarkar & Sewell (2012), "A Tutorial Introduction to the ARM and POWER Relaxed Memory Models."* The accessible entry to weak models. Section 6.4.
- *Alglave, Maranget & Tautschnig (2014), "Herding Cats."* Unified axiomatic framework + `herd` simulator + hardware data-mining methodology; basis of `herdtools7`. Sections 8.2–8.3; core experimental tooling.

**Reference texts and surveys**

- *Nagarajan, Sorin, Hill & Wood (2020), A Primer on Memory Consistency and Cache Coherence, 2nd ed.* The standard modern textbook (open access); adds GPU/accelerator and formal-tools chapters. Used throughout.
- *"Weak Memory Model Formalisms: Introduction and Survey" (2025), arXiv:2508.04115.* Recent consolidation of the formal-models landscape; entry point for the paper's deeper sections. Section 8.4.

**Course texts (primary anchors)**

- *Sarangi, Basic Computer Architecture (v3.09).* Ch. 10 (pipelining/hazards/forwarding); Ch. 11 (memory system); Ch. 12 (multiprocessors), esp. §12.4.2 coherence, §12.4.3 memory consistency, §12.4.7 implementing a consistency model. The introductory on-ramp.
- *Sarangi, Next-Gen Computer Architecture (v2.2).* Ch. 2–4 (out-of-order pipelines, LSQ, commit); Ch. 9 (coherence, consistency, transactional memory), esp. §9.2.2–§9.2.3 (consistency; coherence vs consistency), §9.3 (theoretical foundations, incl. §9.3.2 SC), §9.5 (memory models, incl. §9.5.7 popular models), §9.6 (data races, DRF). The rigorous course treatment.
- *Hennessy & Patterson, Computer Architecture: A Quantitative Approach (5th ed.).* Ch. 5 (thread-level parallelism, coherence, §5.6 models of memory consistency); the benchmarking and quantitative framing. Cross-check reference.
- *Kogge, The Architecture of Pipelined Computers.* The classic on pipelining and overlapped execution — the uniprocessor roots of reordering (Section 5.1).

> **[TABLE 6]** — *Caption:* "Literature map by role and by section." *Description:* A matrix with rows = the sources above, columns = the roles (Foundational / Relaxed-origins / Language / Rigorous-hardware / Tooling / Course-anchor) and the section(s) of this paper that use each. Gives the reader a one-glance bibliographic map.

---

## 10. Discussion: Depth, Breadth, and Open Problems

**On breadth.** The subject spans at least four layers — hardware models (SC — TSO — PSO/RMO — ARM/POWER/Alpha, and now GPU/heterogeneous), language models (Java, C++11, and newer Rust/Go semantics), specification styles (operational, axiomatic), and verification (equivalence proofs, litmus testing, microarchitectural and compiler verification). A term paper can travel a long way horizontally, but the payoff is in choosing a vertical slice and going deep.

**On depth.** Any single model, taken seriously, opens into substantial theory: the equivalence between an operational and an axiomatic presentation is a nontrivial mechanized proof; the "no-thin-air-values" problem in language models remains partly open; multi-copy atomicity has driven real ISA revisions; and the correctness of compiler mappings from language atomics to hardware barriers is its own research literature. The gap between "a two-page definition" (Lamport) and "a mechanized, tested, tool-backed model of a shipping ISA" (x86-TSO, ARMv8) is a good measure of the field's depth.

**Open problems the term paper can point to (or engage with).**

1. **Out-of-thin-air (OOTA) values.** Language models still struggle to forbid absurd self-justifying outcomes in racy/relaxed code without also forbidding legitimate compiler optimizations. Active as of the mid-2020s.
2. **Heterogeneous and scoped consistency.** CPU-GPU systems with scoped synchronization lack a single clean model; correctness across scopes is error-prone.
3. **Persistent memory ordering.** Non-volatile memory adds a *durability* order on top of the visibility order (when does a write survive a crash?), extending the consistency question into a new dimension.
4. **RISC-V (RVWMO) adoption and verification.** A young, open, formally specified model whose ecosystem and hardware conformance are still maturing.
5. **Automated hardware/compiler verification at scale.** Scaling equivalence and conformance checking to full designs remains hard; recent 2025–2026 work (model checking of consistency models) shows the area is active.

**Where our slice sits.** The term paper's chosen depth is the x86-TSO corner: strong enough to reason about cleanly, rigorously specified, well tooled (`herdtools7`), and directly connected to the C++11 model the microbenchmarks target. This lets the paper be concrete and reproducible while still touching the frontier via the survey.

> **[FIGURE 7]** — *Caption:* "Breadth à depth map of the field, with the paper's slice highlighted." *Description:* A 2-D map: horizontal axis = breadth (hardware models — language models — tooling — verification); vertical axis = depth (definition — rigorous model — mechanized proof — conformance). Shade the region the term paper occupies (x86-TSO, C++11, herdtools7 litmus + microbenchmarks) and mark the open problems as points outside it.

---

## 11. Plan for the Remainder of the Term Paper

This draft is the first of an expected ~150-page paper. The remaining sections turn from exposition to investigation. The plan below is written so that this draft ends by opening forward rather than stopping — the later parts are already scoped here.

### 11.1 Research questions

- **RQ1 (Observation).** Which "surprising" memory orderings are actually observable on commodity x86 hardware, and do they match the x86-TSO model exactly? (SB observable; IRIW not; MP safe without a fence; etc.)
- **RQ2 (Cost).** What is the concrete performance cost of each C++11 `memory_order` level on the same workload, and how does it map to the underlying fences the compiler emits?
- **RQ3 (Microarchitecture).** How do microarchitectural parameters (store-buffer depth, speculation, coherence protocol) change which orderings are observable and how expensive ordering is? (gem5, budget permitting.)

### 11.2 Experimental methodology

**Experiment A — Litmus testing on real hardware (primary).**
Using `herdtools7` (`litmus7` to run on hardware; `herd7` to simulate the model), run the canonical suite — SB, MP, LB, IRIW, plus Dekker-style tests — on an x86 machine, collecting outcome histograms over billions of iterations. Cross-check every observed outcome against the `herd7` prediction under the published x86-TSO `.cat` model. Expected result: SB observable, IRIW never observed, MP safe without barriers, confirming TSO. This directly answers RQ1 and operationalizes Sections 6.2 and 8.2.

> **[GRAPH 2]** — *Caption:* "Observed outcome frequencies for SB, MP, LB, IRIW on x86." *Description:* Bar chart per test showing the fraction of runs producing each outcome (including the 'surprising' one), with a marker for whether x86-TSO permits it. Populated from `litmus7` output.

**Experiment B — C++11 atomics microbenchmarks (primary).**
Implement small, representative workloads — a shared counter, a producer/consumer (MP pattern), and a spinlock/handoff — templated over `memory_order` (relaxed, acquire/release, seq_cst). Compile at a fixed optimization level, inspect the emitted assembly to record which fences each variant produces, and measure throughput/latency with Linux `perf` (cycles, instructions, and where available memory-ordering-related stalls). This answers RQ2 and operationalizes Section 7.3.

> **[GRAPH 3]** — *Caption:* "Throughput vs. `memory_order` for each workload." *Description:* Grouped bar chart: x-axis = workload, bars = relaxed / acquire-release / seq_cst, y-axis = throughput (ops/s) or latency; annotate the fence emitted in each case. From `perf` measurements.

> **[TABLE 7]** — *Caption:* "Source `memory_order` — emitted x86 instructions — measured cost." *Description:* Rows = each workloadÃ`memory_order`; columns = source tag, emitted ordering instruction (e.g., plain `mov`, `xchg`, `mfence`), cycles/op, relative cost. Ties source semantics to hardware to `perf` numbers.

**Experiment C — gem5 architectural simulation (optional / stretch).**
If time allows, use gem5 (Binkert et al. 2011) to vary store-buffer depth and coherence/consistency settings and observe the effect on litmus outcomes and on microbenchmark cost, isolating causes that are fixed on real silicon. This addresses RQ3. Flagged optional given its setup complexity.

### 11.3 Structure of the full paper (projected)

1–2. Introduction & background (this draft, expanded). 3. Formal problem and models (this draft's Sections 3–4, deepened with full axiomatic definitions). 4. History & survey (this draft's Sections 5, 9). 5. Hardware model deep-dive with the x86-TSO `.cat` model in full. 6. Language models and the C++11 mapping. 7. Experiment A (litmus/`herdtools7`). 8. Experiment B (C++11 + `perf`). 9. Experiment C (gem5, optional). 10. Discussion, open problems, and conclusions. Appendices: notation, full litmus corpus, raw data.

### 11.4 Risks and mitigations

- *Hardware observability of rare outcomes.* Some legal-but-rare outcomes need careful stressing to observe; mitigation: use `litmus7`'s affinity/stress options and large iteration counts.
- *Compiler variability.* Emitted fences depend on compiler/version/flags; mitigation: pin toolchain versions and archive the emitted assembly alongside results.
- *gem5 scope creep.* Mitigation: keep Experiment C strictly optional and time-boxed; the paper stands on A and B.

---

## 12. Conclusion

On one processor, a program means what it says. On many, it means what the memory consistency model says it means. This draft has traced that shift from its origin — the same pipelining and buffering tricks that speed up a single core, made visible the moment cores share memory — through the reference model that formalizes intuition (Lamport's sequential consistency), the decades of relaxation that traded ordering for speed (TSO, PSO, RMO, ARM, POWER), the migration of the problem into programming languages (Java, C++11), and the modern program of rigorous, tool-backed models (x86-TSO, ARMv8, herding cats). We fixed the coherence/consistency distinction that beginners conflate, stated the problem precisely as a filter on executions, mapped the breadth and depth of the subject, and set out a concrete, reproducible experimental plan grounded in `herdtools7`, C++11 atomics, and — optionally — gem5. Everything here is anchored to the course texts and extended into the primary literature so that the remainder of the term paper can proceed without a discontinuity between what the course teaches and where the research frontier lies. The next installment turns from telling the story to running the experiments.

---

## References

*Convert to a `.bib` during the LaTeX pass. Links are provided where a stable copy exists; DOIs are canonical. Course texts are listed with the specific editions used for section/page references.*

**Primary research literature**

1. L. Lamport. "How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs." *IEEE Transactions on Computers*, C-28(9):690–691, 1979. DOI: 10.1109/TC.1979.1675439. Author copy: https://lamport.azurewebsites.net/pubs/pubs.html ; PDF: https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/How-to-Make-a-Multiprocessor-Computer-That-Correctly-Executes-Multiprocess-Programs.pdf

2. S. V. Adve and K. Gharachorloo. "Shared Memory Consistency Models: A Tutorial." *IEEE Computer*, 29(12):66–76, 1996 (first as WRL Research Report 95/7, 1995). PDF: https://www.cs.cmu.edu/afs/cs/academic/class/15740-f18/www/papers/ieeemicro96-adve-consistency.pdf ; WRL RR: http://rsim.cs.uiuc.edu/arch/qual_papers/arch/adve_shared.pdf

3. M. Dubois, C. Scheurich, and F. Briggs. "Memory Access Buffering in Multiprocessors." *Proc. 13th Int'l Symp. on Computer Architecture (ISCA)*, 1986. DOI: 10.1145/17356.17406

4. S. V. Adve and M. D. Hill. "Weak Ordering — A New Definition." *Proc. 17th ISCA*, 1990. DOI: 10.1145/325164.325100

5. K. Gharachorloo, D. Lenoski, J. Laudon, P. Gibbons, A. Gupta, and J. Hennessy. "Memory Consistency and Event Ordering in Scalable Shared-Memory Multiprocessors." *Proc. 17th ISCA*, 1990. DOI: 10.1145/325164.325102

6. J. Manson, W. Pugh, and S. V. Adve. "The Java Memory Model." *Proc. 32nd ACM POPL*, 2005, pp. 378–391. DOI: 10.1145/1040305.1040336

7. H.-J. Boehm and S. V. Adve. "Foundations of the C++ Concurrency Memory Model." *Proc. ACM PLDI*, 2008, pp. 68–78. DOI: 10.1145/1375581.1375591

8. P. Sewell, S. Sarkar, S. Owens, F. Zappa Nardelli, and M. O. Myreen. "x86-TSO: A Rigorous and Usable Programmer's Model for x86 Multiprocessors." *Communications of the ACM*, 53(7):89–97, 2010. DOI: 10.1145/1785414.1785443. PDF: https://www.cl.cam.ac.uk/~pes20/weakmemory/cacm.pdf ; project: https://www.cl.cam.ac.uk/~pes20/weakmemory/

9. L. Maranget, S. Sarkar, and P. Sewell. "A Tutorial Introduction to the ARM and POWER Relaxed Memory Models." Draft/tutorial, 2012. PDF: https://www.cl.cam.ac.uk/~pes20/ppc-supplemental/test7.pdf

10. J. Alglave, L. Maranget, and M. Tautschnig. "Herding Cats: Modelling, Simulation, Testing, and Data-Mining for Weak Memory." *ACM Transactions on Programming Languages and Systems (TOPLAS)*, 36(2):7, 2014. DOI: 10.1145/2627752

11. V. Vafeiadis, T. Balabonski, S. Chakraborty, R. Morisset, and F. Zappa Nardelli. "Common Compiler Optimisations Are Invalid in the C11 Memory Model and What We Can Do About It." *Proc. ACM POPL*, 2015. DOI: 10.1145/2676726.2676995

12. N. Binkert et al. "The gem5 Simulator." *ACM SIGARCH Computer Architecture News*, 39(2):1–7, 2011. DOI: 10.1145/2024716.2024718

13. (Survey) "Weak Memory Model Formalisms: Introduction and Survey." arXiv:2508.04115, 2025. https://arxiv.org/abs/2508.04115

**Reference textbook**

14. V. Nagarajan, D. J. Sorin, M. D. Hill, and D. A. Wood. *A Primer on Memory Consistency and Cache Coherence*, 2nd ed. Morgan & Claypool (Synthesis Lectures on Computer Architecture), 2020. DOI: 10.2200/S00962ED2V01Y201910CAC049. Open-access PDF: https://library.oapen.org/bitstream/handle/20.500.12657/61248/978-3-031-01764-3.pdf

**Course texts (prescribed)**

15. S. R. Sarangi. *Basic Computer Architecture*, v3.09. 2025. https://srsarangi.github.io/archbook/archbook.pdf — used: Ch. 10 (pipelining), Ch. 11 (memory system), Ch. 12 (multiprocessors; §12.4.2 coherence, §12.4.3 memory consistency, §12.4.7 implementing a consistency model; §12.1 Moore's law; §12.2.2 shared memory vs message passing).

16. S. R. Sarangi. *Next-Gen Computer Architecture: Till the End of Silicon*, v2.2. https://www.cse.iitd.ac.in/~srsarangi/advbook/nextgen-comp-arch.pdf — used: Ch. 2–4 (out-of-order pipelines, LSQ §4.3, commit §4.4), Ch. 9 (§9.2.2–§9.2.3 consistency and coherence-vs-consistency, §9.3 theoretical foundations incl. §9.3.2 SC and §9.3.4 PLSC→coherence and §9.3.5 SC via synchronisation, §9.5 memory models incl. §9.5.7 popular models, §9.6 data races incl. §9.6.3–§9.6.4 DRF).

17. J. L. Hennessy and D. A. Patterson. *Computer Architecture: A Quantitative Approach*, 5th ed. Morgan Kaufmann, 2011 — used: Ch. 5 (thread-level parallelism; §5.4 coherence; §5.6 models of memory consistency). Course-listed copy: https://allbooksfordownloading.wordpress.com/wp-content/uploads/2017/01/computer-architecture-a-quantitative-approach-by-hennessy-and-patterson-5th-edition.pdf

18. P. M. Kogge. *The Architecture of Pipelined Computers*. McGraw-Hill, 1981 — used: pipelining and overlapped/out-of-order execution as the uniprocessor origin of memory reordering (Section 5.1). (Listed in the course as 1977; the widely cited McGraw-Hill edition is 1981 — verify the year against the course syllabus before final submission.)

---

*End of first draft. Placeholders marked [FIGURE], [TABLE], [GRAPH], [DIAGRAM] indicate intended non-text components to be produced during typesetting; the prose is self-contained without them. Section/page cross-references to Sarangi's volumes are keyed to versions v3.09 (Basic) and v2.2 (Next-Gen).*
