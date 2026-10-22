# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A Claude Code AI guide that suggests and applies style improvements to an Ada codebase (see README.md). The deliverable is guidance for Claude itself, grounded in the official Ada style guides and language rationales collected under `corpus/`.

## Current state

There is no source code, build system, or test suite yet. The repository contains only `README.md`, `LICENSE` (MIT), `.gitignore`, and the `corpus/` reference material. `.gitignore` already excludes GNAT build products (`*.o`, `*.ali`), so any Ada code added later is expected to be built with GNAT/gprbuild.

`corpus/` is **untracked** in git. Treat it as read-only source material; do not edit the documents in it.

## Corpus layout

`corpus/<year>/{style guide,rationale}/` holds one Ada language revision per year. Formats differ by year, so pick the right tool before reading:

| Year | Style guide | Rationale |
|------|-------------|-----------|
| 1983 | *Ada '83 Quality and Style* as HTML chapters (`sty-NN-MM.html`, `index.html`, `sty-TOC.html`) | *Ada '83 Rationale* as HTML chapters (`ratl-NN-MM.html`, `ratl-00.html`) |
| 1995 | *Ada 95 Quality and Style* (DoD AJPO), single PDF `95style.pdf`, 304 pages | HTML (`rat95-pN-M.html`, `rat95-contents.html`) |
| 2005 | Wikibooks *Ada Style Guide* PDF | *Ada 2005 Rationale* HTML (`Rat-N-M.html`, `Rat-TOC.html`) |
| 2012 | Wikibooks *Ada Style Guide* PDF | *Ada 2012 Rationale* HTML (`Rat12-N-M.html`, `Rat12-TOC.html`) plus the Technical Corrigendum rationale (`RCorr-N.html`, `RCorr-TOC.html`) |
| 2022 | empty | *Overview of Ada 2022* PDF (`Ada2022-Overview.pdf`, 57 pages) |

Notes:
- The 2005 and 2012 style guide PDFs are byte-identical copies of the Wikibooks Ada Style Guide (March 2013 build). Reading one covers both.
- The 1983 and 1995 "Quality and Style" guides are the authoritative, detailed style references; the Wikibooks PDF is lighter.
- HTML rationale files are one section per file. Start from the TOC/contents file for a year to find the section you want.

## Reading the corpus

`pdftotext` and `pdfinfo` (poppler) are available. Extract a page range rather than a whole book:

```sh
pdftotext -f 50 -l 60 "corpus/1995/style guide/95style.pdf" -
```

For the HTML sections, grep across a year and strip tags as needed:

```sh
grep -ril "naming" corpus/1995/rationale
sed 's/<[^>]*>//g' corpus/1983/style\ guide/sty-03-01.html
```

Directory names contain a space (`style guide`), so quote paths.
