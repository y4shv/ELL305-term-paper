# Project context

This file is read automatically at the start of every Claude Code session.
It carries the context established when this paper was planned, so that neither
Claude nor a new collaborator has to be re-briefed.

---

## What this is

A term paper for **ELL305 Computer Architecture** at IIT Delhi.

**Title:** Revisiting Sequential Consistency: Memory Consistency Models, Their
Cost, and Whether Speculation Has Made Strong Ordering Affordable

**Deadlines:** four submissions. v0.1, v0.2, v0.3 are drafts (6 marks each),
stored on GitHub and submitted via Gradescope. v1.0 is final (12 marks), due
**11 November 2026**.

**Target length:** v0.1 is 30–40 pages. v1.0 is approximately 150 pages /
35,000 words of body, plus roughly 50 pages of appendices.

---

## The argument

The paper is not a survey. It has a thesis, and every chapter serves it.

> Store buffers make writes fast but break the ordering programmers assume.
> Cache coherence does not fix this, because the store buffer sits upstream of
> the coherence protocol. The industry's answer was to weaken the architectural
> contract and hand programmers fences. That decision was made around 1990 on
> in-order, non-speculative cores with fewer than sixteen processors. A modern
> out-of-order core already has both mechanisms a strong model needs — coherence
> invalidations to detect remote writes, and a squash path built for branch
> misprediction. This paper asks whether the performance objection to sequential
> consistency still holds, and if it does not, what else explains the industry's
> continued preference for weak ordering.

Four questions structure the work:

- **Q1 Formal** — express SC, TSO, PSO, weak ordering and RC as constraints on
  one set of ordering relations; prove strict containment. (Ch. 4)
- **Q2 Empirical** — which relaxations are *observable* on real x86 and ARM
  hardware, as distinct from merely *permitted*? (Ch. 6–7)
- **Q3 Quantitative** — holding microarchitecture fixed and varying only the
  memory model, what does SC cost with and without speculation, as core count
  rises? (Ch. 8–9)
- **Q4 Interpretive** — if speculative SC is cheap, why does nobody ship it?
  (Ch. 11)

**Do not let the paper drift into a summary of the textbook.** Chapters 6–10
are roughly 40% of the body and exist only if the experiments are actually run.
That is the difference between a term paper and a long book report.

---

## Current state

**v0.1 is complete: 42 pages, compiles clean, zero undefined references.**

| File | Contents |
|---|---|
| `chapters/00-title.tex` | Title page — **placeholders still to fill in** |
| `chapters/00-abstract.tex` | Abstract |
| `chapters/01-introduction.tex` | SB example, problem statement, contributions, textbook coverage table |
| `chapters/02-background.tex` | Memory wall, caches, store buffer, out-of-order |
| `chapters/03-coherence.tex` | MSI/MESI/MOESI, coherence vs consistency |
| `chapters/04-models.tex` | Relations, litmus tests, five models, containment theorem, fences, DRF |
| `chapters/05-history.tex` | 1979 → RVWMO chronology |
| `chapters/06-methodology.tex` | Experimental design for v0.2/v0.3 |
| `chapters/07-sources.tex` | Full source specification with reproduced ToC extracts |

### Planned for v0.2 and v0.3

- Ch. 7 → renumber; Results I: litmus tests on real x86 and ARM
- Ch. 8: Simulator design
- Ch. 9: Results II: SC vs TSO vs Weak, with and without speculation
- Ch. 10: Verification — axiomatic checker vs simulator
- Ch. 11: Discussion — does the 1990 verdict still hold?
- Ch. 12: Conclusion
- Appendices A–G: litmus catalogue, proofs, simulator docs, raw data, fence
  reference, protocol state tables, annotated bibliography

---

## Conventions — follow these

**One sentence per line.** Git merges line by line. A paragraph on one long line
means any two edits collide; one sentence per line means they merge silently.
LaTeX ignores single newlines, so output is unaffected. The existing files
already follow this. Keep it.

**Formatting is fixed by the course.** A4, 1 inch margins, 0.5 inch binding
offset, 11 pt, single spaced. Do not change the geometry.

**All figures in TikZ or circuitikz.** No external images, no PNG, no
matplotlib output pasted in as a graphic — plots for the results chapters are
generated as vector PDF from CSV, but structural diagrams are TikZ. There is a
house style defined in `main.tex` (`unit`, `core`, `buf`, `mem`, `flow`, `slow`,
`lbl`). Use it rather than inventing per-figure styles.

**`refs.bib` is append-only.** Never sort or reformat it — a reformat touches
every line and conflicts with everything the other author did.

**`main.pdf` is gitignored.** It is binary; git cannot merge it. Build locally.
For submissions, tag the commit and attach the PDF to a GitHub Release.

**Notation.** Follows Sarangi's *Next-Gen* §9.4 (execution witnesses, access
graphs) rather than the Primer's or Alglave's, for continuity with the course.
Macros are defined in `main.tex`: `\po`, `\rf`, `\co`, `\fr`, `\ppo`, `\Ld{}`,
`\St{}`, `\code{}`. A correspondence table is in §7.6.

---

## Sources

Full specification with reproduced ToC extracts is in `chapters/07-sources.tex`.
In brief:

- **Sarangi, *Basic Computer Architecture* v3.09** — prescribed. Primary
  coverage is §12.4; supported by §11.2, §11.3, §10.4, §10.9, §10.11.4, §12.2,
  §12.6, Appendix A.
- **Sarangi, *Next-Gen Computer Architecture*** (White Falcon 2023) — Ch. 9 is
  ~118 pages on exactly this topic. Source of the formalism.
- **Nagarajan, Sorin, Hill & Wood, *Primer on Memory Consistency and Cache
  Coherence*, 2nd ed.** — the standard reference.
- Hennessy & Patterson Ch. 5; Culler, Singh & Gupta Ch. 5, 6, 8, 9; Herlihy et
  al. 2nd ed.
- Vendor manuals: Intel SDM Vol. 3A, ARM ARM Ch. B2, RISC-V RVWMO, Linux LKMM.
  **Cite the manual, not a textbook, for any claim about what a specific ISA
  guarantees.**

Page numbers for Next-Gen and the Primer are deliberately omitted from Ch. 7 —
they differ between printings and must be verified against the author's own
copy before citing.

---

## Build

```bash
latexmk -pdf main.tex
```

If `lmodern.sty` or `microtype` is missing, install
`texlive-fonts-recommended`, or comment out those two lines in `main.tex`.
Nothing else depends on them.

---

## Working practices

**Scope every edit.** This repo has two authors with a file-ownership split (see
`COLLABORATING.md`). Do not edit files outside the scope of the request. If
asked to "improve the paper", ask which chapter.

**Do not invent citations.** If a claim needs a source and none is in
`refs.bib`, say so rather than fabricating an entry. Every reference must be one
the author can actually produce.

**Do not pad.** The page target is met by running the experiments, not by
lengthening prose. If a chapter is short, the answer is usually more data, not
more words.

**The author is new to this material.** They are learning computer architecture
while writing about it, and they will be examined on this text in a viva.
Prefer explaining a change to silently making it. Prose the author has not read
is worse than no prose.

---

## Immediate to-do

1. Fill in name, entry number, instructor in `chapters/00-title.tex`
2. Fill in the ownership table in `COLLABORATING.md`
3. Verify Sarangi section numbers against the author's copy of v3.09
4. Write and run the SB litmus test; put the observed violation count into §1.1
   so the opening is a measurement rather than an assertion
