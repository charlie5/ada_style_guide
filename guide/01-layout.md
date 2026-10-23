# 1. Layout and Source Code Presentation

Rules `LAY-*`. Source: Ada 95 Quality and Style (Q&S95) chapter 2; Ada 83 Quality and Style (Q&S83) chapter 2.

The single most important layout rule is consistency across a compilation unit and across the project. Where a project already follows a different but consistent convention (for example 2-space indentation or a 100-column limit), respect the project's convention and report only inconsistencies. The numbers below are the guides' recommended defaults, offered when a project has no convention of its own.

Almost every rule in this file is mechanical and safe to apply with a formatter. Prefer running the project's formatter (for GNAT, `gnatpp`; GNAT's `-gnaty` switches check most of these) over hand-editing. Two things a formatter cannot judge are called out: semantic alignment of expression terms and blank-line grouping.

---

## LAY-01 Spacing around delimiters and operators

**Rule.** Use consistent spacing around delimiters, following ordinary prose: a space on both sides of binary operators and `:=`, `=>`, `|`, `..`, `:`, `<>`; a space after commas and semicolons; spaces outside but not inside parentheses. No space after unary `+`/`-`, before a call's opening parenthesis, inside `<< >>`, around `**`, `'` and `.`, between consecutive parentheses, or before `,` and `;`.

**Why.** Delimiters are one or two characters and get lost among keywords and identifiers; surrounding white space makes them stand out. Spacing punctuation differently from prose is distracting.

**Detect.** `auto`. Regexes on code outside strings and comments: `\S:=`, `:=\S`, `\S=>`, `=>\S`, `,\S`, `\s,`, `\s;`, `\(\s`, `\s\)`, `\w\s+\(` on a call (allowed only for alignment).

**Apply.** `auto`. Formatter or regex edit. Note the stated exception: when precedence makes parentheses unnecessary, spaces may be dropped around the highest-precedence operators (`2*X + 1`).

**Example.**
```ada
Register (Index) := Memory (Base_Address + Index * Element_Length);   -- Q&S writes Register(Index); either is fine, be consistent
Error_Term := 1.0 - (Cos (Theta)**2 + Sin (Theta)**2);
type Signed_Whole_16 is range -2**15 .. 2**15 - 1;
```

**Source.** Q&S95 §2.1.1; Q&S83 §2.1.1.

---

## LAY-02 Indentation

**Rule.** Indent nested control structures, continuation lines and embedded units consistently. Use a different step for nesting and for continuation lines so the two are visually distinct. Indent with spaces, never tabs. Recommended defaults: 3 spaces per nesting level, 2 spaces for continuation lines, labels outdented 3 spaces. Use the paragraphing shown in the Ada Reference Manual syntax rules (`if`/`elsif`/`else`/`end if` aligned; `loop` body indented; `record` on its own line under `type ... is`, components indented under it; `when` choices of a `case` indented one level, their statements another; `private` aligned with `package`).

**Why.** Indentation is the reader's visual indicator of structure; it lets the first and last keyword of a construct be matched by eye. Tabs render differently on different devices. Consistency matters more than the number of spaces.

**Detect.** `auto`. Tab characters: `grep -P '\t'`. Mixed indentation widths: compare leading-space counts of `begin`/`end` pairs and of `if`/`end if` pairs. GNAT `-gnaty3` checks 3-space indentation.

**Apply.** `auto` via formatter. If the project uses a different consistent step (commonly 2 or 4), keep it.

**Example.**
```ada
Outer :
   for I in Items'Range loop
      if Input_Found then
         Count_Characters;
      else --  not Input_Found
         Reset_State;
         Character_Total :=
           First_Part_Total * First_Part_Scale_Factor +
           Second_Part_Total * Second_Part_Scale_Factor;
      end if;
   end loop Outer;
```

**Source.** Q&S95 §2.1.2 (layout tables for every construct); Q&S83 §2.1.2; ARM §1.1.4.

---

## LAY-03 Vertical alignment of operators

**Rule.** Align operators vertically in consecutive related statements (typically `:=` in a block of assignments). In a long expression, break lines at semantic boundaries so related terms line up, and leave room for trailing comments on each term.

**Why.** Alignment puts visual emphasis on what the code is doing and exposes the shape of an expression.

**Detect.** `assisted`. Runs of 3+ consecutive assignments whose `:=` columns differ.

**Apply.** `assisted`. Alignment is safe to apply, but do not force it if it pushes a statement over the line limit or breaks it at a bad spot (the guide's stated exception). Warn before running a formatter on hand-aligned expressions: most formatters destroy semantic line breaks and collect the comments at the end.

**Example.**
```ada
Temporary := Slot_A;
Slot_A    := Slot_B;
Slot_B    := Temporary;

Y := (A * B + C) + (2.0 * D - E) -   -- basic equation
     3.5;                            -- account for error factor
```

**Source.** Q&S95 §2.1.3; Q&S83 §2.1.3.

---

## LAY-04 Alignment of declarations, one declaration per line

**Rule.** At most one declaration per line. All declarations in one declarative part at the same indentation. For declarations not separated by blank lines, align the `:` delimiters, the `:=` delimiters and the `--` of trailing comments, producing a table. When a declaration overflows, break (in order of preference) at the comment, at `:=`, or at `:`, and indent the wrapped part. Enumeration literals that do not fit on one line go one per line, or in rows/columns that group semantically related literals.

**Why.** The tabular layout gives compiler and reader the same information and removes the need for redundant "variable dictionary" header comments that drift out of date. Names stop hiding in a mass of declarations.

**Detect.** `auto` for multiple declarations on one line (`^\s*\w+\s*:.*;\s*\w+\s*:` or `A, B : T` is fine, but `A : T; B : T` on one line is not). `assisted` for misaligned colons in a run.

**Apply.** `auto`. A formatter aligns colons; a checker should accept the tabular enumeration layout even though a formatter would reflow it.

**Example.**
```ada
Prompt_Column : constant        := 40;
Question_Mark : constant String := " ? ";   -- prompt on error input
Prompt_String : constant String := " ==> ";

type Op_Code is
  (Push,            Pop,
   Add,             Subtract,          Multiply,       Divide,
   Subroutine_Call, Subroutine_Return,
   Branch,          Branch_On_Zero,    Branch_On_Negative);
```

**Source.** Q&S95 §2.1.4; Q&S83 §2.1.4.

---

## LAY-05 Parameter lists: one parameter per line, aligned modes

**Rule.** One formal parameter specification per line. Vertically align parameter names, colons, the words `in` and `out`, and the subtypes, so mode columns line up (`in     `, `    out`, `in out`). Put the first parameter on the same line as the subprogram name; if that forces the subtypes past the line limit, start the list on a new continuation line. Align parentheses vertically in complex conditions so the terms of the expression are visible.

**Why.** Produces a table with columns for name, mode, subtype and per-parameter comment. Vertical alignment across the subprograms of a unit helps even more.

**Detect.** `auto`. Several parameters on one line of a specification: `\(\s*\w+\s*:\s*[^;)]+;\s*\w+\s*:`.

**Apply.** `auto`. Stated exception: an operator function whose parameters share a type reads better on one line: `function "&" (Left, Right : Color_Scheme) return Color_Scheme;`. Also keep short one-parameter specifications on one line.

**Example.**
```ada
procedure Display_Menu (Title   : in     String;
                        Options : in     Menus;
                        Choice  :    out Alpha_Numerics);

if not (First_Character in Alpha_Numerics and then
        Valid_Option (First_Character))        then
```

**Source.** Q&S95 §2.1.5; Q&S83 §2.1.5.

---

## LAY-06 Blank lines group related text

**Rule.** Use blank lines to separate logically related groups: between kinds of declarations, between a type and its subtypes, around nested control structures, between subprograms.

**Why.** Consistent, deliberate blank lines make sections of related code visible.

**Detect.** `assisted`. Long declarative parts with no blank lines; several consecutive blank lines (more than two is noise).

**Apply.** `assisted`. The decision is semantic. Formatters cannot place these, though most preserve existing ones.

**Source.** Q&S95 §2.1.6; Q&S83 §2.1.6.

---

## LAY-07 Pagination: mark unit boundaries

**Rule.** Highlight the top of each package or task specification, the top of each program-unit body, and the `end` of each unit. Use the file prologue and unit headers of [COM-02](03-comments.md) for the outer unit. For nested units in a declarative part, put a line of dashes (starting at the current indentation) immediately before and after the unit; when two dashed lines would be adjacent, keep only one.

**Why.** Parts of a unit not visible on the current screen are easy to overlook. Marked boundaries let a reader confirm the whole unit is visible and scan a large file quickly.

**Detect.** `auto`. Package bodies with several nested subprograms and no separator lines between them.

**Apply.** `auto`. Insert separators matching the project's existing separator style, if any. Do not insert separators where a project deliberately uses none; report instead.

**Example.**
```ada
   ---------------------------------------------------------------
   function Max (Left, Right : Tiny_Integer) return Tiny_Integer is
   begin
      ...
   end Max;
   ---------------------------------------------------------------
   function Min (Left, Right : Tiny_Integer) return Tiny_Integer is
```

**Source.** Q&S95 §2.1.7; Q&S83 §2.1.7.

---

## LAY-08 One statement per line

**Rule.** Start each statement on a new line, write no more than one simple statement per line, and break compound statements (`if`, `loop`, `case`, `select`) over multiple lines. A statement longer than the line continues on the next line. Preferred line-break points are after semicolons (ARM §1.1.4). This includes declarations, context clauses and parameters.

**Why.** One statement per line lets the reader find statements and not miss any; the structure of a compound statement is clearer with its parts on separate lines.

**Detect.** `auto`. Two statement-terminating semicolons on one line outside strings/comments: `;\s*\S.*;` (exclude `for`/parameter lists). `then\s+\S.*;` for one-line `if`s.

**Apply.** `auto`. Stated exception: closely related short statement groups laid out as a table, for instance a column of `Put ("A="); Natural_IO.Put (A); New_Line;` lines. Leave such tables alone.

**Example.**
```ada
--  Not:  if End_Of_File then Close_File; else Get_Next_Record; end if;
if End_Of_File then
   Close_File;
else
   Get_Next_Record;
end if;
```

**Source.** Q&S95 §2.1.8; Q&S83 §2.1.8.

---

## LAY-09 Maximum line length

**Rule.** Adhere to a maximum line length. The guide's instantiation is 72 characters, with 79 as the stated alternative (avoids the 80th column on terminals, and distinguishes the limit from Fortran's 72). GNAT's default style check uses 79. Modern projects commonly choose 100 or 120; the point is a hard, enforced limit.

**Why.** Portability of source files between systems and printers, publishability on letter paper, and the human field of view for reading code, which is roughly 70 to 80 columns.

**Detect.** `auto`. `awk 'length > N'` on each file; GNAT `-gnatyM<n>`.

**Apply.** `auto`-detect, `assisted`-fix: re-break long lines at semantic boundaries ([LAY-03](#lay-03-vertical-alignment-of-operators)), after commas, or after semicolons, keeping continuation indentation.

**Source.** Q&S95 §2.1.9; Q&S83 §2.1.9; Q&S95 §7.1.2 note on portability.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| LAY-01 | Prose-like spacing around delimiters and operators | auto |
| LAY-02 | Consistent indentation, spaces not tabs, distinct continuation step | auto |
| LAY-03 | Align operators; break long expressions at semantic boundaries | assisted |
| LAY-04 | One declaration per line, tabular alignment of `:` `:=` `--` | auto |
| LAY-05 | One parameter per line, aligned modes | auto |
| LAY-06 | Blank lines group related text | assisted |
| LAY-07 | Mark unit boundaries | auto |
| LAY-08 | One statement per line; compound statements multi-line | auto |
| LAY-09 | Enforced maximum line length | auto |
