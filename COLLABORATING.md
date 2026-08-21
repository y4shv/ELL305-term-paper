# Collaborating on this paper

Two people editing one LaTeX document is where student projects usually break.
The conventions below exist to make that not happen. They take five minutes to
learn and save a great deal of pain later.

---

## 1. One-time setup

### Person A — create the repository

On github.com: **New repository** → name it something like
`ell305-term-paper` → set it **Private** → do *not* tick "Add a README"
(we already have one).

Then, from the project folder:

```bash
git init
git add .
git commit -m "v0.1 draft: chapters 1-7, bibliography, TikZ figures"
git branch -M main
git remote add origin https://github.com/<your-username>/ell305-term-paper.git
git push -u origin main
```

Now add your partner: repository → **Settings** → **Collaborators** →
**Add people** → their GitHub username. They accept by email.

### Person B — clone it

```bash
git clone https://github.com/<person-A-username>/ell305-term-paper.git
cd ell305-term-paper
```

### Both — set your identity and verify the build

```bash
git config user.name  "Your Name"
git config user.email "your@email.com"

latexmk -pdf main.tex     # or: pdflatex main && bibtex main && pdflatex main && pdflatex main
```

If `lmodern.sty` or `microtype` is missing, install
`texlive-fonts-recommended` — or comment those two lines out of `main.tex`.
Nothing else depends on them.

---

## 2. The daily rhythm

Four commands. Learn these and you will rarely have trouble.

```bash
git pull            # ALWAYS first. Before you open the editor.
# ... do your work ...
git add -A
git commit -m "Ch 4: tighten the containment proof"
git push
```

The rule that matters: **`git pull` before you start, `git push` when you
stop.** Most conflicts come from someone editing a stale copy for three hours.

### Should we use branches and pull requests?

For two people on a term paper, no. A shared `main` with the discipline above
is simpler and faster. Branches earn their keep when you need review gates or
have five contributors; you have neither.

The one exception: if one of you is doing something large and disruptive —
restructuring a chapter, say — branch for that:

```bash
git checkout -b restructure-ch4
# work, commit, push
git push -u origin restructure-ch4
```

Then open a pull request on GitHub so the other person can look before it
lands on `main`.

---

## 3. Avoiding conflicts

### Own your chapters

The single most effective measure. Agree who owns which files and stay out of
each other's. The paper is deliberately split one chapter per file for exactly
this reason.

| File | Owner |
|---|---|
| `chapters/01-introduction.tex` | |
| `chapters/02-background.tex` | |
| `chapters/03-coherence.tex` | |
| `chapters/04-models.tex` | |
| `chapters/05-history.tex` | |
| `chapters/06-methodology.tex` | |
| `chapters/07-sources.tex` | |
| `main.tex` | *both — announce before editing* |
| `refs.bib` | *both — append only, never reorder* |

Fill in the names. Revisit the split when new chapters appear in v0.2.

Two files need care because both of you will touch them. For `main.tex`, say
so in chat first — edits are rare and small. For `refs.bib`, **only ever add
entries at the end**. Never sort or reformat it; a reformat touches every line
and conflicts with everything your partner did.

### One sentence per line

The source files follow this convention, and you should keep writing that way:

```latex
The SWMR invariant is the operative one.
It partitions the lifetime of each line into epochs, alternating between a
single exclusive writer and a set of concurrent readers.
```

Not this:

```latex
The SWMR invariant is the operative one. It partitions the lifetime of each line into epochs, alternating between a single exclusive writer and a set of concurrent readers.
```

Git merges line by line. When a paragraph is one long line, any two edits
anywhere in that paragraph collide. When each sentence is its own line, you can
both edit the same paragraph and git merges it silently. This costs nothing —
LaTeX ignores single newlines — and it is the difference between occasional
conflicts and constant ones.

It also makes diffs readable, so you can actually see what your partner
changed.

---

## 4. When a conflict happens anyway

Git will mark the file like this:

```
<<<<<<< HEAD
your version of the sentence
=======
their version of the sentence
>>>>>>> origin/main
```

Open the file, decide what the text should say, delete all three marker lines,
then:

```bash
git add <file>
git commit
git push
```

**Compile before you push.** A conflict resolution that leaves a stray
`>>>>>>>` in the source produces a LaTeX error your partner will inherit.

If it goes badly wrong and you want to start over from what's on GitHub:

```bash
git stash          # parks your changes safely, does not delete them
git pull
git stash pop      # brings them back on top
```

---

## 5. The PDF

`main.pdf` is in `.gitignore`, deliberately. It is a binary file; git cannot
merge two versions of it, so committing it means a conflict on nearly every
pull. Build it locally instead — it takes seconds.

For the four course submissions, tag the commit and attach the PDF to a GitHub
Release:

```bash
git tag -a v0.1 -m "Draft v0.1 as submitted"
git push origin v0.1
```

Then on GitHub: **Releases** → **Draft a new release** → choose the `v0.1` tag
→ upload `main.pdf`.

This gives you a permanent, dated record of exactly what was submitted, which
is worth having when a marked draft comes back and you need to see what the
comments referred to.

---

## 6. Working with Claude Code

Both of you can point Claude Code at your own clone. It is a normal git
repository and nothing special is needed.

What does *not* happen: your two Claude sessions do not see each other. There
is no shared state between them. Coordination is entirely through git — your
partner sees your work when you push and they pull, and not before.

The practical implication is that the file-ownership table above matters more,
not less, when you are both working with an agent. An agent asked to "improve
the paper" may touch files well outside its lane. Be specific about scope:

> "Only edit `chapters/04-models.tex`. Do not touch other chapters or
> `refs.bib`."

And review the diff before committing:

```bash
git diff
```

That habit is worth forming regardless. You will be defending this text in a
viva, and you cannot defend a paragraph you have not read.

---

## 7. Before each submission

- [ ] `git pull`, then a clean build from scratch (`latexmk -C && latexmk -pdf main.tex`)
- [ ] Zero LaTeX errors; check for undefined references in the log
- [ ] Title page has both names and entry numbers
- [ ] Every claim traceable to a source you have actually read
- [ ] Both of you can explain every chapter, not just your own
- [ ] Tag the commit and attach the PDF to a Release
- [ ] Submit on Gradescope
