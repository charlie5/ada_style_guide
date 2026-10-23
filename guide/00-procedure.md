# 0. Review Procedure

How to use this guide to suggest and apply style improvements to an Ada codebase. The procedure has five stages: inventory, project profile, detection, report, apply. Stages 1 to 4 are read-only. Stage 5 changes code and runs only within the scope the user has asked for.

The guide's rules are in the numbered files; [README.md](README.md) explains the rule format and the three apply levels (`auto`, `assisted`, `advisory`).

---

## 1. Inventory

Establish what is being reviewed before reading any rule.

1. **Find the sources.** `find . -name '*.ads' -o -name '*.adb' -o -name '*.ada' -o -name '*.gpr'` plus any project-specific naming (`*.1.ada`/`*.2.ada`, `*_spec.ada`). Count units and lines. Note generated code (exclude it) and third-party code vendored into the tree (exclude it unless asked).
2. **Find the build.** GNAT project files (`*.gpr`), `alire.toml`, Makefiles. Record the compiler switches: language version (`-gnat2005`, `-gnat2012`, `-gnat2022`, `-gnatX`, or `pragma Ada_20xx` in `gnat.adc`), style checks (`-gnaty...`), warnings (`-gnatw...`), assertion policy (`-gnata`), check suppression (`-gnatp`). Record whether a formatter is configured (`gnatpp` switches, a `.gnatpp` or `gnatformat` configuration) and whether `gnatcheck` rules exist.
3. **Find existing conventions.** `CLAUDE.md`, `CONTRIBUTING`, `STYLE`, `docs/coding-standard*`, header comments of a few representative files. A written project standard overrides this guide's defaults wherever the two conflict; the guide then checks conformance to that standard and still reports the rules the standard does not cover.
4. **Decide the language version.** From step 2. If nothing states it, assume Ada 2012, say so in the report, and ask the user to confirm before applying any `MOD-*` rule.
5. **Can it be built?** Try the project's documented build command (for GNAT, `gprbuild -q -p -P project.gpr` or `alr build`). A working build is the safety net for every applied change. If it cannot be built here, every change stays at "proposed" unless the user says otherwise.

## 2. Project profile

Measure the project's actual conventions on a sample of files (a dozen spread across directories, plus the largest and the oldest) and write them down. The guide's numbers are defaults; the project's consistent choices win ([README.md](README.md), Precedence).

| Convention | How to measure | Guide default |
|------------|----------------|---------------|
| Indentation step | leading spaces of `begin` after `procedure`/`package body` | 3 |
| Continuation indent | indentation of lines after an unterminated statement | 2 |
| Line length | `awk '{ if (length > m) m = length } END { print m }'`, and the 95th percentile | 72 or 79 |
| Tabs | `grep -lP '\t'` | none |
| Identifier casing | sample of `type`, `procedure`, object declarations | `Mixed_Case`, lower-case reserved words |
| Acronym casing | upper-case runs in identifiers | upper case, from a project list |
| Type naming | `_Type`/`_T` suffix frequency versus plain nouns | plain nouns |
| Spacing before `(` in calls | `\w\(` versus `\w \(` counts | either, consistently |
| `use` clauses | count of `use Pkg;` in specs and bodies | minimal; `use type` |
| Prefixed notation | `Obj.Op (` on tagged objects versus `Pkg.Op (Obj,` | project choice |
| Overriding indicators | fraction of overriding primitives marked | all (Ada 2005+) |
| Named association | fraction of 3+-argument calls using `=>` | yes |
| Header comment style | first comment block of specs | file header plus unit headers |
| Separator lines between units | `^\s*-{20,}` | dashed lines |
| Aspects versus pragmas | `pragma Inline` versus `with Inline` | aspects (Ada 2012+) |

Where a convention is consistent (say above 90 percent), adopt it as the project's rule and report only the deviations. Where it is split, report the split and ask which to standardise on; do not pick one silently. Record the profile at the top of the report.

## 3. Detection

Work rule file by rule file, or by the subset the user asked for. For each rule use its **Detect** entry:

- Run the grep patterns across the sources. They are heuristics: strings, comments and generated code produce false positives, so confirm each hit by reading the surrounding lines before it becomes a finding.
- Use the compiler where it does the job better: `-gnatwa` (all warnings), `-gnatyy` (default style checks), `-gnatwu` (unused withs), `-gnatVa` (validity), `-gnatwm`/`-gnatwk` (modes and constants). Compile once with these switches and map the warnings to rule IDs.
- Use `gnatcheck` if the project has it; many `LAY`, `NAM` and `STM` rules have direct equivalents.
- For `advisory` rules, read rather than grep: package specs for interface size and cohesion, type hierarchies for OOP rules, task bodies for concurrency rules.

Keep per-finding notes as you go: `file:line`, rule ID, the offending text, and for `auto` rules the exact replacement.

## 4. Report

Produce the report before changing anything, unless the user asked to apply directly (then still produce it, after). Structure:

1. **Scope and profile.** What was reviewed, the language version, the build status, the project profile table, and any written standard found.
2. **Summary table.** One row per rule with findings: rule ID, one-line title, count, apply level, proposed action (apply / propose / report).
3. **Findings by rule**, most valuable first: correctness-adjacent rules (`EXC`, `CON`, `TYP` Part C, `POR` obsolescent features) before layout. Each finding is `path:line` plus a one-line statement and, for `auto`/`assisted` rules, the replacement. Collapse mass findings (hundreds of spacing hits) into counts per file with a few examples.
4. **Convention questions** the user must answer: split conventions, acronym list, whether to adopt prefixed notation or `@`, whether assertions are enabled in production (affects `MOD-13`), whether portability is a goal (affects how strictly `POR` rules apply).
5. **Available on upgrade.** `MOD` rules gated out by the project's language version, with counts.
6. **Not reviewed.** Excluded directories, generated code, anything skipped for size.

Keep the prose short; the tables carry the content.

## 5. Apply

Change code only within the scope the user asked for, and in this order:

1. **`auto` rules first**, one rule (or one closely related group such as all `LAY`) per batch. After each batch: rebuild; run the test suite if there is one; diff-check that only the intended lines changed. A build failure after an `auto` change means either the heuristic misfired or the code was already broken; investigate, do not pile on further changes.
2. **`assisted` rules** next, each finding confirmed by reading the code. Renames follow the rename protocol below. Where the rule needs a project decision that has not been given, leave it in the report.
3. **`advisory` rules** are never applied unasked. When the user asks for one, treat it as a design change: plan it, confirm the plan, implement it in isolation, build and test.

Rules of conduct while applying:

- **Do not run a whole-file formatter unless the project already uses one** with a checked-in configuration. A formatter run rewrites every line and buries the real changes; it also destroys semantic alignment ([LAY-03](01-layout.md)) and tabular enumeration layouts ([LAY-04](01-layout.md)). Prefer targeted edits.
- **One concern per commit** when committing is requested: layout, naming, a modernisation rule. Commit messages name the rule IDs applied.
- **Rename protocol.** Before renaming any declaration: search the whole project (including named parameter associations, `renames`, child unit names, file names under the GNAT naming scheme, generic actuals, comments and documentation) and list every site. Rename every site in one batch. Rebuild. Public names of a library consumed by other projects are renamed only with the user's agreement.
- **Comments and headers** that a rule removes or rewrites must not lose information that is not elsewhere; move it rather than delete it.
- **Never change behaviour** while applying a style rule. If the correct fix changes behaviour (a missing `Close`, a swallowed exception, a shared variable without protection), stop and report it as a defect, not a style finding.
- **Verify, then say so.** Report each batch as built-and-tested, built-only, or unverified, with the output of any failure.

## 6. When to stop and ask

Ask the user, in one message, when: the language version is unknown; a convention is split and the rule cannot proceed without a choice; a finding looks like a defect rather than style; an `auto` change breaks the build for a reason that is not obvious; a rename touches a published interface; or the review scope would exceed what was asked (for instance, a layout rule that would touch every file when the user asked about one package).
