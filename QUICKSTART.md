# Quickstart

## Person A — create and push the repo (do this once)

```bash
git init
git add .
git commit -m "v0.1 draft"
git branch -M main
git remote add origin https://github.com/<YOUR-USERNAME>/ell305-term-paper.git
git push -u origin main
```

Create the repo on github.com first: **New repository**, name it
`ell305-term-paper`, set it **Private**, and do NOT tick "Add a README" or
"Add .gitignore" — both are already here.

Then add your partner: **Settings → Collaborators → Add people**.

## Person B — clone it (once)

```bash
git clone https://github.com/<PERSON-A-USERNAME>/ell305-term-paper.git
cd ell305-term-paper
git config user.name  "Your Name"
git config user.email "your@email.com"
```

## Both — every working session

```bash
git pull                    # before you open the editor
# ... work ...
git add -A
git commit -m "Ch 4: tightened the containment proof"
git push                    # when you stop
```

## Build the PDF

```bash
latexmk -pdf main.tex
```

`main.pdf` is gitignored on purpose — build it locally. If `lmodern.sty` or
`microtype` is missing, install `texlive-fonts-recommended`, or comment out
those two lines in `main.tex`. Nothing else depends on them.

## Before you submit v0.1

1. Fill in name, entry number and instructor in `chapters/00-title.tex`
2. Fill in the ownership table in `COLLABORATING.md`
3. Read the paper. Do not submit text you cannot defend.
4. Verify the Sarangi section numbers against your copy of v3.09
5. Tag and release:
   ```bash
   git tag -a v0.1 -m "Draft v0.1 as submitted"
   git push origin v0.1
   ```
   Then attach `main.pdf` to a GitHub Release.

Read `COLLABORATING.md` before you both start editing — especially the
one-sentence-per-line convention. It is what keeps merges painless.
