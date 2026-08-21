# ELL305 Term Paper — v0.1

**Title:** Revisiting Sequential Consistency: Memory Consistency Models, Their
Cost, and Whether Speculation Has Made Strong Ordering Affordable

## Build

    pdflatex main && bibtex main && pdflatex main && pdflatex main

Or simply `latexmk -pdf main.tex`.

Formatting follows the course specification exactly: A4, 1 in margins, 0.5 in
binding offset (left gutter), 11 pt, single spaced. All figures are TikZ. All
references live in `refs.bib`, exported from Zotero.

## Layout

    main.tex              preamble, geometry, TikZ house style, macros
    refs.bib              bibliography
    chapters/
      00-title.tex        title page  -- FILL IN name, entry number, instructor
      00-abstract.tex
      01-introduction.tex motivating example, problem statement, contributions
      02-background.tex   memory wall, caches, store buffer, out-of-order
      03-coherence.tex    MSI/MESI/MOESI, coherence vs consistency
      04-models.tex       formal relations, litmus tests, SC/TSO/PSO/Weak/RC
      05-history.tex      1979 -> RVWMO chronology
      06-methodology.tex  experimental design for v0.2 and v0.3
      07-sources.tex      full specification of every text used, with
                          reproduced ToC extracts (course form Q12)

## Still to do before submitting v0.1

1. Fill in name / entry number / instructor on the title page.
2. Read the sources and make the prose your own -- do not submit text you
   cannot defend in a viva.
3. Verify every claim against the cited source. Page numbers for the Sarangi
   citations should be checked against your copy of v3.09.

## Planned structure for v1.0 (~150 pp body + ~50 pp appendices)

    Ch 7   Results I  -- litmus tests on real x86 and ARM hardware
    Ch 8   Simulator design
    Ch 9   Results II -- SC vs TSO vs Weak, with and without speculation
    Ch 10  Verification -- axiomatic checker vs simulator
    Ch 11  Discussion -- does the 1990 verdict still hold?
    Ch 12  Conclusion
    App A  Litmus test catalogue with source
    App B  Proofs (containment, DRF)
    App C  Simulator documentation
    App D  Raw data tables
    App E  Fence reference across ISAs
    App F  MSI / MESI / MOESI transition tables
    App G  Annotated bibliography
