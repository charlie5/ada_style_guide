# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A Claude Code AI guide that suggests and applies style improvements to an Ada codebase (see README.md). The deliverable is guidance for Claude itself, grounded in the official Ada style guides and language rationales collected under `corpus/`.

## Layout

- `guide/` is the deliverable: a rule-based Ada style guide written for Claude Code. `guide/README.md` explains the rule format, apply levels and sources; `guide/00-procedure.md` is the review procedure; `guide/01-*.md` to `guide/15-*.md` hold the rules (`LAY`, `NAM`, `COM`, `STR`, `TYP`, `STM`, `SUB`, `EXC`, `CON`, `POR`, `REU`, `OOP`, `PRF`, `MOD`, `RAT`); `guide/rules-index.md` is generated from the checklist tables at the end of each rule file.
- `.claude/skills/ada-style/SKILL.md` is the skill that drives a review; invoke it with `/ada-style` or by asking for an Ada style review.
- `corpus/` is the source material (untracked in git). Treat it as read-only; do not edit the documents in it.
- There is no Ada source code, build system or test suite in this repository. `.gitignore` excludes GNAT build products (`*.o`, `*.ali`) for any Ada code added later.

## Maintaining the guide

- Every rule keeps the entry shape described in `guide/README.md` (Rule, Why, Detect, Apply, Example, Source) and a stable ID. Add new rules at the end of a file's numbering; do not renumber.
- Each rule file ends with a `## Checklist` table (`| ID | Rule | Apply |`; `14-modern-ada.md` has an extra `Since` column). Regenerate `guide/rules-index.md` after editing a checklist:

  ```sh
  python3 tools/build_rules_index.py
  ```

  The script also fails on duplicate IDs and on a checklist ID with no matching `## ID` heading.
- Cite sources as the README's "Cited as" column shows (`Q&S95 §5.6.3`, `Rat 2012 §3.5`, `AARM22 Annex J`). Section numbers refer to the original documents; the Wikibooks PDF uses shifted numbers and is not cited.
- Rules in `14-modern-ada.md` must state the Ada revision that introduced the feature; the procedure gates them on the project's language version.

## Corpus layout

`corpus/<year>/{style guide,rationale}/` holds one Ada language revision per year (2022 also has a `language reference manual (annotated)/` directory). Formats differ by year, so pick the right tool before reading:

| Year | Style guide | Rationale |
|------|-------------|-----------|
| 1983 | *Ada '83 Quality and Style* as HTML chapters (`sty-NN-MM.html`, `index.html`, `sty-TOC.html`) | *Ada '83 Rationale* as HTML chapters (`ratl-NN-MM.html`, `ratl-00.html`) |
| 1995 | *Ada 95 Quality and Style* (DoD AJPO), single PDF `95style.pdf`, 304 pages | HTML (`rat95-pN-M.html`, `rat95-contents.html`) |
| 2005 | Wikibooks *Ada Style Guide* PDF | *Ada 2005 Rationale* HTML (`Rat-N-M.html`, `Rat-TOC.html`) |
| 2012 | Wikibooks *Ada Style Guide* PDF | *Ada 2012 Rationale* HTML (`Rat12-N-M.html`, `Rat12-TOC.html`) plus the Technical Corrigendum rationale (`RCorr-N.html`, `RCorr-TOC.html`) |
| 2022 | empty | *Overview of Ada 2022* PDF (`Ada2022-Overview.pdf`, 57 pages); plus `language reference manual (annotated)/` holding the Ada 2022 Annotated Reference Manual as HTML sections (`AA-N-M.html`, `AA-TOC.html`, Annex J obsolescent features under `AA-J-*.html`) |

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

Directory names contain spaces and parentheses (`style guide`, `language reference manual (annotated)`), so quote paths.
