# Ada Style Guide for Claude Code

A rule-based guide that lets Claude Code **suggest and apply style improvements to an Ada codebase**. It distils the authoritative Ada style literature into numbered rules, each with a detection heuristic and an instruction for how safely it can be applied.

Start with [00-procedure.md](00-procedure.md), which says how to run a review. The rules live in the numbered files below; [rules-index.md](rules-index.md) lists every rule on one page.

## Files

| File | Rules | Covers |
|------|-------|--------|
| [00-procedure.md](00-procedure.md) | | How to review and apply: inventory, project profile, detection, findings, apply protocol |
| [01-layout.md](01-layout.md) | `LAY` | Spacing, indentation, alignment, blank lines, pagination, line length |
| [02-naming.md](02-naming.md) | `NAM` | Spelling, casing, abbreviations, names for types, objects, units, constants, exceptions |
| [03-comments.md](03-comments.md) | `COM` | File, specification and body headers; data, statement and marker comments |
| [04-structure.md](04-structure.md) | `STR` | Files and units, child packages, cohesion, interfaces, visibility, `use` and `renames` |
| [05-types.md](05-types.md) | `TYP` | Declaring types, data structures, dynamic data, unchecked features, initialisation |
| [06-statements.md](06-statements.md) | `STM` | Names on loops and ends, expressions, control structures, aggregates |
| [07-subprograms.md](07-subprograms.md) | `SUB` | Parameters, named association, defaults, modes, overloading, equality |
| [08-exceptions.md](08-exceptions.md) | `EXC` | Exceptions in interfaces and in bodies |
| [09-concurrency.md](09-concurrency.md) | `CON` | Protected objects, tasks, communication, termination |
| [10-portability.md](10-portability.md) | `POR` | Obsolescent features, platform assumptions, numerics, representation, I/O |
| [11-reusability.md](11-reusability.md) | `REU` | Reusable parts and generics |
| [12-object-oriented.md](12-object-oriented.md) | `OOP` | Tagged types, dispatching, controlled types, constructors, multiple inheritance |
| [13-performance.md](13-performance.md) | `PRF` | Measured-performance alternatives and pragmas |
| [14-modern-ada.md](14-modern-ada.md) | `MOD` | Ada 2005, 2012 and 2022 features that replace older idioms; upgrade incompatibilities |
| [15-rationale-advice.md](15-rationale-advice.md) | `RAT` | Usage advice stated in the Ada 83 and Ada 95 Rationales |
| [rules-index.md](rules-index.md) | | One-line index of every rule |

## Rule entries

Every rule has the same shape:

- **ID** such as `STM-07`, stable across edits of the guide.
- **Rule.** What to do, close to the source's wording, with the source's stated exceptions.
- **Why.** The source's own rationale, so a finding can explain itself.
- **Detect.** How to find violations: grep patterns, compiler switches, or "manual". Patterns are heuristics over code outside strings and comments and will produce some false positives; the reviewer confirms each hit.
- **Apply.** One of three levels, plus any caveat:
  - `auto`: an exact transformation that changes no semantics; make it, then recompile.
  - `assisted`: the change is well defined but needs a reading of intent, a project decision, or an update of every use site; make it when confident, otherwise propose it with the edit spelled out.
  - `advisory`: a design-level change; report it with a concrete suggestion and do not make it unasked.
- **Example.** Before/after where the source gives one.
- **Source.** Section references into the corpus.

## Precedence

1. **The project's own consistent convention beats the guide's default.** Where a project is consistent (2-space indentation, `_Type` suffixes, prefixed notation everywhere), the guide's job is to find the inconsistencies, not to impose its numbers. The guide says so itself: "Above all, be consistent across your entire project."
2. **Readability beats performance unless a measurement says otherwise** ([13-performance.md](13-performance.md)).
3. **Language version gates the modern rules.** Nothing from [14-modern-ada.md](14-modern-ada.md) is applied to code compiled with an earlier standard; it is reported as available on upgrade.
4. **Portability rules tighten other rules only when portability is a stated goal.**

## Sources

The corpus is under `corpus/` in this repository, by language revision.

| Source | Cited as | Role |
|--------|----------|------|
| *Ada 95 Quality and Style: Guidelines for Professional Programmers* (SPC, 1995) | Q&S95 §x.y | Primary source for files 01 to 13. Every guideline bullet of chapters 2 to 10 is represented. |
| *Ada '83 Quality and Style* (SPC, 1992) | Q&S83 §x.y | Predecessor with the same section structure; cited for continuity. Its Ada 83 workarounds (no child units, no controlled types, rendezvous-only sharing) are not repeated. |
| Wikibooks *Ada Style Guide* (2013 build; `corpus/2005` and `corpus/2012`) | not cited | On inspection this is the Ada 95 guide reprinted with section numbers shifted by one and the exception chapters omitted; it adds no rules. |
| *Rationale for the Design of the Ada Programming Language* (1986) | Ada 83 Rationale §x.y | File 15. |
| *Ada 95 Rationale* (Barnes et al.) | Ada 95 Rationale §x.y | File 15, and cited throughout where the Q&S guide itself cites it. |
| *Rationale for Ada 2005* (Barnes) | Rat 2005 §x.y | File 14, Part A. |
| *Rationale for Ada 2012* (Barnes) and the Corrigendum update | Rat 2012 §x.y, RCorr §n | File 14, Part B. |
| *Overview of Ada 2022* (Cousins, version 121) | Ada 2022 Overview §x.y | File 14, Part C. |
| *Ada 2022 Annotated Reference Manual* (draft 33) | ARM / AARM22 §x.y, Annex J | Language rules cited by the style rules; the current obsolescent-feature list. |

To read a source section, see `CLAUDE.md` at the repository root for the file layout and extraction commands.

## Using the guide from another project

The guide is plain Markdown and self-contained. Either reference it by path from the other project's `CLAUDE.md` ("Ada style rules: `/path/to/ada_style_guide/guide/`; follow `00-procedure.md`"), or copy the `.claude/skills/ada-style` directory of this repository into the other project and set its `guide` path. The skill is a thin wrapper that loads the procedure and the rule files on demand.
