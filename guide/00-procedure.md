# 0. Review Procedure

How to use this guide to suggest and apply style improvements to an Ada codebase. The procedure has five stages: inventory, project profile, detection, report, apply. Stages 1 to 4 are read-only. Stage 5 changes code and runs only within the scope the user has asked for.

The guide's rules are in the numbered files; [README.md](README.md) explains the rule format and the three apply levels (`auto`, `assisted`, `advisory`).

---

## 1. Inventory

Establish what is being reviewed before reading any rule.

1. **Find the sources.** `find . -name '*.ads' -o -name '*.adb' -o -name '*.ada' -o -name '*.gpr'` plus any project-specific naming (`*.1.ada`/`*.2.ada`, `*_spec.ada`). Count units and lines. Note generated code (exclude it) and third-party code vendored into the tree (exclude it unless asked).
2. **Find the build.** GNAT project files (`*.gpr`), `alire.toml`, Makefiles. Record the compiler switches: language version (`-gnat2005`, `-gnat2012`, `-gnat2022`, `-gnatX`, or `pragma Ada_20xx` in `gnat.adc`), style checks (`-gnaty...`), warnings (`-gnatw...`), assertion policy (`-gnata`), check suppression (`-gnatp`). Record whether a formatter is configured (`gnatpp` switches, a `.gnatpp` or `gnatformat` configuration) and whether `gnatcheck` rules exist.
3. **Find existing conventions.** `CLAUDE.md`, `CONTRIBUTING`, `STYLE`, `docs/coding-standard*`, header comments of a few representative files, and any style or formatting guide the project keeps in its tree or points to from those files. A written project standard overrides this guide's defaults wherever the two conflict; the guide then checks conformance to that standard and still reports the rules the standard does not cover.
   - Read the standard in full before detection starts. For each rule of this guide that the standard contradicts, add a line to an **overrides list**: rule ID, the standard's section, and what the standard prescribes instead. The list goes into the report (stage 4, item 4) and governs stage 5: a rule on it is never applied `auto`.
   - Example: a project following the lace formatting guide (`ada_formatting_guide/guide/`) leaves most loops and all blocks anonymous, writes no marker comments, instantiates ubiquitous generics positionally, keeps aligned `pragma import` runs, and declares one-off tables as anonymous arrays. Its overrides list names [STM-01](06-statements.md), [STM-02](06-statements.md), [COM-07](03-comments.md), the generic-instantiation clause of [SUB-02](07-subprograms.md), [TYP-04](05-types.md), and the pragma-to-aspect conversions of [MOD-12](14-modern-ada.md) and [POR-01](10-portability.md).
4. **Decide the language version.** From step 2. If nothing states it, assume Ada 2012, say so in the report, and ask the user to confirm before applying any `MOD-*` rule.
5. **Can it be built?** Try the project's documented build command (for GNAT, `gprbuild -q -p -P project.gpr` or `alr build`). A working build is the safety net for every applied change. If it cannot be built here, every change stays at "proposed" unless the user says otherwise.

## 2. Project profile

Measure the project's actual conventions on a sample of files (a dozen spread across directories, plus the largest and the oldest) and write them down. The guide's numbers are defaults; the project's consistent choices win ([README.md](README.md), Precedence).

| Convention | How to measure | Guide default |
|------------|----------------|---------------|
| Indentation step | leading spaces of `begin` after `procedure`/`package body` | none; measured only for the nesting heuristic of [STM-09](06-statements.md) |
| Type naming | `_Type`/`_T` suffix frequency versus plain nouns | plain nouns |
| `use` clauses | count of `use Pkg;` in specs and bodies | minimal; `use type` |
| Prefixed notation | `Obj.Op (` on tagged objects versus `Pkg.Op (Obj,` | project choice |
| Overriding indicators | fraction of overriding primitives marked | all (Ada 2005+) |
| Named association | fraction of 3+-argument calls using `=>` | yes |
| Header comment style | first comment block of specs | unit headers |
| Aspects versus pragmas | `pragma Inline` versus `with Inline` | aspects (Ada 2012+) |

Where a convention is consistent (say above 90 percent), adopt it as the project's rule and report only the deviations. Where it is split, report the split and ask which to standardise on; do not pick one silently. Record the profile at the top of the report.

## 3. Detection

Work rule file by rule file, or by the subset the user asked for. For each rule use its **Detect** entry:

- Run the grep patterns across the sources. They are heuristics: strings, comments and generated code produce false positives, so confirm each hit by reading the surrounding lines before it becomes a finding.
- Use the compiler where it does the job better: `-gnatwa` (all warnings), `-gnatwu` (unused withs), `-gnatVa` (validity), `-gnatwm`/`-gnatwk` (modes and constants), plus the project's own style switches (`-gnaty...`) recorded in stage 1 step 2. Do not add style letters the project leaves off: `-gnatyy` turns on casing, line-length and token-spacing checks that a house style may deliberately reject, and the noise buries the real findings. Compile once with these switches and map the warnings to rule IDs.
- The grep patterns in the **Detect** entries are written in the Reference Manual's casing (`Constraint_Error`, `Ada.`, `pragma Inline`, `_Type`). When the project profile shows any other casing scheme (lower-case prefixes such as `ada.`, head-word casing such as `constraint_Error`), run every pattern case-insensitively (`grep -i`) and confirm the hits by reading, or the review silently misses most of what it is looking for.
- Use `gnatcheck` if the project has it; many `NAM` and `STM` rules have direct equivalents.
- For `advisory` rules, read rather than grep: package specs for interface size and cohesion, type hierarchies for OOP rules, task bodies for concurrency rules.

Keep per-finding notes as you go: `file:line`, rule ID, the offending text, and for `auto` rules the exact replacement.

## 4. Report

Produce the report before changing anything, unless the user asked to apply directly (then still produce it, after). Structure:

1. **Scope and profile.** What was reviewed, the language version, the build status, the project profile table, and any written standard found.
2. **Summary table.** One row per rule with findings: rule ID, one-line title, count, apply level, proposed action (apply / propose / report).
3. **Findings by rule**, most valuable first: correctness-adjacent rules (`EXC`, `CON`, `TYP` Part C, `POR` obsolescent features) first. Each finding is `path:line` plus a one-line statement and, for `auto`/`assisted` rules, the replacement. Collapse mass findings (hundreds of hits for one `auto` rule) into counts per file with a few examples.
4. **Conflicts with the project standard.** The overrides list from stage 1 step 3: one row per rule, with the guide's position, the standard's position and its section, and the number of sites the rule would have touched. Findings for these rules appear here, not under item 3, so the user decides once per rule rather than once per finding. Omit the section when the project has no written standard.
5. **Convention questions** the user must answer: split conventions, acronym list, whether to adopt prefixed notation or `@`, whether assertions are enabled in production (affects `MOD-13`), whether portability is a goal (affects how strictly `POR` rules apply).
6. **Available on upgrade.** `MOD` rules gated out by the project's language version, with counts.
7. **Not reviewed.** Excluded directories, generated code, anything skipped for size.

Keep the prose short; the tables carry the content.

## 5. Apply

Change code only within the scope the user asked for, and in this order:

1. **`auto` rules first**, one rule (or one closely related group) per batch, excluding every rule on the overrides list: a rule the project standard contradicts is applied, at most as `assisted`, only after the user has chosen the guide's side over the standard's, and is otherwise left in the report. After each batch: rebuild; run the test suite if there is one; diff-check that only the intended lines changed. A build failure after an `auto` change means either the heuristic misfired or the code was already broken; investigate, do not pile on further changes.
2. **`assisted` rules** next, each finding confirmed by reading the code. Renames follow the rename protocol below. Where the rule needs a project decision that has not been given, leave it in the report.
3. **`advisory` rules** are never applied unasked. When the user asks for one, treat it as a design change: plan it, confirm the plan, implement it in isolation, build and test.

Rules of conduct while applying:

- **Do not run a whole-file formatter unless the project already uses one** with a checked-in configuration. A formatter run rewrites every line and buries the real changes. Prefer targeted edits.
- **One concern per commit** when committing is requested: naming, comments, a modernisation rule. Commit messages name the rule IDs applied.
- **Rename protocol.** Before renaming any declaration: search the whole project (including named parameter associations, `renames`, child unit names, file names under the GNAT naming scheme, generic actuals, comments and documentation) and list every site. Rename every site in one batch. Rebuild. Public names of a library consumed by other projects are renamed only with the user's agreement.
- **Comments and headers** that a rule removes or rewrites must not lose information that is not elsewhere; move it rather than delete it.
- **Never change behaviour** while applying a style rule. If the correct fix changes behaviour (a missing `Close`, a swallowed exception, a shared variable without protection), stop and report it as a defect, not a style finding.
- **Verify, then say so.** Report each batch as built-and-tested, built-only, or unverified, with the output of any failure.

## 6. When to stop and ask

Ask the user, in one message, when: the language version is unknown; a convention is split and the rule cannot proceed without a choice; a finding looks like a defect rather than style; an `auto` change breaks the build for a reason that is not obvious; a rename touches a published interface; or the review scope would exceed what was asked (for instance, an `auto` rule that would touch every file when the user asked about one package).
