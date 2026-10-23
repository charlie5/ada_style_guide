---
name: ada-style
description: Review an Ada codebase against the Ada style guide in guide/ and suggest or apply style improvements. Use when asked to review, check, lint, clean up, modernise or apply style rules to Ada sources (.ads/.adb), or when asked about Ada style, naming, layout, comments, exceptions, tasking, portability or Ada 2005/2012/2022 modernisation.
---

# Ada style review

This skill drives the rule-based guide in `guide/` (relative to this repository's root; if the skill has been copied into another project, set `GUIDE` below to the guide directory's absolute path).

```
GUIDE = guide
```

## Steps

1. Read `$GUIDE/00-procedure.md` in full. It defines the five stages (inventory, profile, detection, report, apply) and the rules of conduct. Follow it.
2. Read `$GUIDE/README.md` for the rule format, the apply levels (`auto`, `assisted`, `advisory`) and the precedence rules. The project's own consistent convention beats the guide's defaults.
3. Decide the scope from the user's request:
   - "review" / "check" / "what would you change": stages 1 to 4 only; produce the report.
   - "apply" / "fix" / "clean up": stages 1 to 5; apply `auto` rules, confirm `assisted` ones individually, leave `advisory` ones in the report.
   - A named area ("naming", "exceptions", "modernise to Ada 2012"): load only the matching rule files (see the table in `$GUIDE/README.md`) plus `01-layout.md` if layout is implicated.
   - Default when unspecified: the whole guide, report only.
4. Load rule files lazily: `$GUIDE/rules-index.md` to pick rules, then the full entry in the numbered file for any rule you detect or apply. Do not paste rule text into the report; cite the ID and give the finding.
5. Gate `$GUIDE/14-modern-ada.md` on the project's language version as the procedure says; never apply a feature newer than the project compiles with.
6. Build after every batch of applied changes. Report build and test results faithfully.

## Output

The report format is in `$GUIDE/00-procedure.md` section 4. Lead with the summary table. Findings are `path:line  RULE-ID  one line`, grouped by rule, most valuable rules first. End with the questions the user must answer and the list of rules available on language upgrade.
