# 6. Expressions and Statements

Rules `STM-*`. Source: Ada 95 Quality and Style (Q&S95) §5.1 Optional Parts of the Syntax, §5.5 Expressions, §5.6 Statements; Q&S83 §5.1, §5.5, §5.6.

Many of these are purely syntactic and safe to apply. The ones that restructure control flow (`STM-11` onwards) must be compiled and, where tests exist, tested after each change.

---

## STM-01 Name nested loops and any loop with an exit

**Rule.** Give a name to every loop that is nested inside another loop, and to every loop containing an `exit`. Use the loop name on every `exit` from a nested loop (`exit Outer when ...`). A loop that is hard to name probably needs more thought.

**Why.** The name is required on `end loop`, so the reader can match ends across a screen boundary; an `exit` is an implicit goto and should say what it leaves; a later inserted loop cannot silently capture an unnamed `exit`.

**Detect.** `auto`. A `loop` whose body contains another `loop` and which has no `Name :` before it; `^\s*exit\s+when` or `^\s*exit\s*;` inside a nested loop without a loop name.

**Apply.** `auto`. Add `Name :` before the loop, `Name` after its `end loop`, and the name on each `exit` belonging to it. Pure syntax; compile to confirm.

**Source.** Q&S95 §5.1.1, §5.1.3.

---

## STM-02 Name nested blocks

**Rule.** Give a name to a `declare` block nested inside another block (and to any long block). Repeat it after `begin` as a comment and on `end`.

**Detect.** `auto`. Unnamed `declare` directly inside another `declare` or a block more than a screen long.

**Apply.** `auto`.

**Source.** Q&S95 §5.1.2.

---

## STM-03 Repeat the name on every `end`

**Rule.** Write `end Unit_Name;` for every package spec and body, subprogram body, task spec and body, protected unit, entry `accept`, and (via `STM-01`/`STM-02`) every named loop and block.

**Detect.** `auto`. `^\s*end\s*;` where the construct is a unit, `accept` or named loop. GNAT `-gnatye` checks this.

**Apply.** `auto`. Compiler-verified.

**Source.** Q&S95 §5.1.4.

---

## STM-04 'First, 'Last, 'Range and the subtype name instead of literals

**Rule.** Express range bounds with `'First`/`'Last` rather than numeric literals or the constants used in the type's declaration; express a whole range with the subtype name (`for I in Weather_Stations loop`) or `'Range`, never `T'First .. T'Last`. For arrays use the object's own attributes (`A'First`, `A'Last`, `A'Length`, `A'Range`), never the index subtype's name, so the code survives a change of bounds or index type.

**Why.** The loop and the offset calculation stay correct if the type changes. Caution from the guide: make sure the right subtype is named. `array (Large_Range)` where `Large_Range is new Integer` asks for a 2^32-element array.

**Detect.** `auto`. `for \w+ in 1 \.\. \w+\b` where the bound names an array's length constant; `'First \.\. \w+'Last` on the same subtype; `for I in Index_Subtype loop` where the body indexes an array whose index subtype may differ; `\(\s*1\s*\)` indexing of arrays whose first index is not literally 1 (strings from `Ada.Strings` functions start wherever they like).

**Apply.** `auto` for `T'First .. T'Last` to `T'Range` or `T`; `assisted` for literal bounds, since `1 .. N` on a `String` slice may be deliberate.

**Example.**
```ada
for I in File_Path'Range loop        --  not: for I in 1 .. Name_Length loop
Offset := Current_Temperature - Temperature'First;
```

**Source.** Q&S95 §5.5.1, §5.5.2; §3.2.6.

---

## STM-05 Parentheses to clarify and to force evaluation order

**Rule.** Add parentheses when an uncommon mix of operators would make the reader check precedence (`5 + ((Y ** 3) mod 10)`), and whenever correctness depends on left-to-right evaluation among operators of equal precedence, the rule most often overlooked. Omit them where Ada's algebraic precedence makes the meaning obvious (`2*I + 4*Y + 8*Z + C`).

**Detect.** `assisted`. Expressions mixing `mod`/`rem` with `*` or `**` without parentheses; mixed `and`/`or` is already illegal without them.

**Apply.** `auto`. Adding parentheses that match the existing parse is safe.

**Source.** Q&S95 §5.5.3; ARM §4.5.

---

## STM-06 Positive logic

**Rule.** Name flags so they are tested in positive form, and write conditions positively: `if Operator_Missing then`, not `if not Operator_Found then` and certainly not `if not Operator_Missing then`. Put the most frequently taken branch first. Stated exception: when the negative expresses the real condition better, keep it.

**Detect.** `auto`. `if not \w+ then` followed by `else`; identifiers starting `Not_`, `No_`, `Dont_`, `Is_Not_`; `/= True`, `= False`.

**Apply.** `assisted`. Swapping the branches of an `if not X then A else B end if` to `if X then B else A end if` is safe; renaming a flag needs every use updated.

**Source.** Q&S95 §5.5.4.

---

## STM-07 Short-circuit forms where the second operand depends on the first

**Rule.** Use `and then` / `or else` whenever the first condition guards the evaluation of the second (`Y /= 0 or else X / Y /= 10`; `Target /= null and then Target.Distance < Threshold`). Plain `and`/`or` give no evaluation order and no guarantee of early exit. If it matters that all operands are always evaluated, a function has a side effect, which violates [STR-05](04-structure.md).

**Detect.** `auto`. `/=\s*null\s+and\s+` (not `and then`); `and\s+\w+\s*\(` where the second operand indexes or dereferences; `/=\s*0\s+and\s+.*\/`.

**Apply.** `auto` where the left operand is a null or zero or range test guarding the right. Elsewhere `assisted`: switching to short-circuit changes nothing when operands are pure, and is a bug fix when they are not.

**Source.** Q&S95 §5.5.5.

---

## STM-08 No equality tests on real operands

**Rule.** Compare fixed- and floating-point values with `<=`, `>=`, `<`, `>`, or against a tolerance, never with `=` or `/=`, and never use `=` as a loop exit condition on a real. Stated exception: a deliberate test of exact representation.

**Detect.** `auto`. `=\s*[0-9]+\.[0-9]` and `exit when .* = .*` where the operand is of a real type.

**Apply.** `assisted`. Replace with `>=`, `<=`, or `abs (A - B) <= Epsilon`; the choice of epsilon needs the application.

**Source.** Q&S95 §5.5.6; §7.2.7.

---

## STM-09 Limit nesting depth

**Rule.** Keep expression and control-structure nesting to at most five levels (three to five is the recommended band). To flatten: hoist a subexpression into a constant; extract a nested region into a subprogram; convert nested `if`s into a `case`; use `elsif` instead of `else if`; reorder the conditions; or reconsider the design. Invert a leading `if not C then ... else ... end if` so the positive branch comes first.

**Detect.** `auto`. Indentation depth: lines indented more than 5 × indent-step inside one subprogram; `else\s+if\b` on one line (should be `elsif`); `if` whose first branch is `not`.

**Apply.** `auto` for `else if ... end if; end if;` to `elsif`; `assisted` for the rest.

**Example.**
```ada
if Condition_1 then
   Action_C;
elsif Condition_2 then
   Action_A;
else --  not (Condition_1 or Condition_2)
   Action_B;
end if;
```

**Source.** Q&S95 §5.6.1.

---

## STM-10 Slices instead of element-copy loops

**Rule.** Copy or assign part of an array with a slice assignment, not with a loop over the elements.

**Detect.** `auto`. `for I in .* loop\s+\w+\s*\(I.*\)\s*:=\s*\w+\s*\(I.*\);\s*end loop`.

**Apply.** `auto`. `Dst (A .. B) := Src (C .. D);` with matching lengths. Check Q&S95 §10.5.7 if the project is performance-critical on a compiler that implements slices badly (rare today).

**Source.** Q&S95 §5.6.2.

---

## STM-11 Case statements: complete, explicit, preferred over if/elsif chains

**Rule.** Use `case` rather than `if`/`elsif` when selecting on one discrete value. List every alternative explicitly; avoid `when others`, so that adding a literal to the type is a compile error rather than a silently absorbed value. Do not use ranges of enumeration literals (`when Red .. Blue`) for the same reason; if a range is needed, declare a named subtype where it is logically part of the abstraction and use the subtype name as the choice. `when others` is acceptable when the subtype is huge (`Character`, `Integer`), and required for a generic formal type; if the interesting values are few compared with the range, an `if`/`elsif` may be clearer. Prefer dispatching to `case` when the selection is on an object's kind ([TYP-09](05-types.md)).

**Detect.** `auto`. `if \w+ = \w+ then` followed by two or more `elsif \w+ = ` on the same variable; `when \w+ \.\. \w+ =>` with enumeration bounds; `when others =>` in a `case` on an enumeration type.

**Apply.** `auto` for `if`/`elsif` on a discrete variable with literal comparisons to `case` (when the `else` branch, if any, becomes `when others`, flag it); `assisted` for removing `others`, because the alternatives for the remaining literals must be written. Expanding `when Red .. Blue` to `when Red | Green | Blue` is safe and exact.

**Source.** Q&S95 §5.6.3; §10.5.3.

---

## STM-12 Choose the loop form by the termination structure

**Rule.** Use `for` whenever the iteration range can be expressed up front (it cannot be infinite and the counter cannot be modified). Use `while` when the number of iterations is unknown but a simple continuation test applies at the top. Use a plain `loop` with `exit` for everything else: loop-and-a-half, repeat-until, multiple exit conditions. Do not put `exit` in a `for` or `while` loop; a reader who has seen the complete condition at the top does not expect another. Minimise the number of exits; more than two is a sign the algorithm wants restructuring.

**Detect.** `auto`. `exit` inside `for`/`while` loops; `while True loop`, `while not Done loop` with `Done := True` inside; flag variables that exist only to end a loop; `for` loops whose body modifies the data structure to force termination.

**Apply.** `assisted`. Converting `while not Done loop ... Done := True ... end loop` to a named plain loop with `exit Name when ...` is mechanical; converting a `for` with an early `exit` into a plain loop with a counter is a judgement call, and in Ada 2012 a quantified expression or a `for ... of` may do it better.

**Example.**
```ada
P_And_Q_Processing :
   loop
      P;
      exit P_And_Q_Processing when Condition_Dependent_On_P;
      Q;
   end loop P_And_Q_Processing;
```

**Source.** Q&S95 §5.6.4, §5.6.5.

---

## STM-13 `exit when`, not `if ... then exit`

**Rule.** Write `exit Name when Condition;` rather than `if Condition then exit; end if;` unless other statements must also run conditionally. Prefer two sequential `exit when` statements to one with a complex short-circuit condition. Review where the exits sit: many scattered exits mean the loop's purpose is fuzzy, and a series of loops may be clearer.

**Detect.** `auto`. `if .* then\s*exit[^;]*;\s*end if;`.

**Apply.** `auto`. Exact transformation.

**Source.** Q&S95 §5.6.5.

---

## STM-14 Bound loops and recursion in safety-critical code

**Rule.** For loops that are not `for` loops, and for recursion, consider a safety counter with a realistic bound that raises an exception when exceeded. This is a safety net, not control logic. Deliberately infinite loops in embedded control software are the exception and should be documented as a policy decision. For non-critical systems the guide calls this optional.

**Detect.** `assisted`. `while \w+ /= null loop` over linked structures; recursive subprograms with no depth parameter. Report only when the project is safety-related.

**Apply.** `advisory`.

**Source.** Q&S95 §5.6.6.

---

## STM-15 No goto

**Rule.** Do not use `goto`. Ada's loop exits and exceptions cover the classic uses. If one must remain, comment at the label where the gotos come from.

**Detect.** `auto`. `\bgoto\b`, `<<\w+>>`.

**Apply.** `assisted`. Replace with a named loop and `exit`, a `return`, or restructuring. (Ada 2022 code sometimes uses `goto Continue;` to emulate a loop `continue`; a named plain loop or an `if` is the guide's answer.)

**Source.** Q&S95 §5.6.7.

---

## STM-16 Few return statements, made visible

**Rule.** Minimise `return` statements in a subprogram. Where several are needed, put them at the same level (the arms of one `case` or `if`) rather than scattered through nested code. A Boolean function with `return True`/`return False` in nested branches should return the condition itself. Highlight remaining returns with a comment. Do not contort the structure merely to reach a single return.

**Detect.** `auto`. More than three `return` statements in one body; `return True;` and `return False;` both present; `return` at nesting depth 3 or more.

**Apply.** `auto` for `if C then return True; else return False; end if;` to `return C;`; `assisted` otherwise.

**Example.**
```ada
return Pointer /= null and then Pointer.Count > 0;
```

**Source.** Q&S95 §5.6.8.

---

## STM-17 Blocks to localise declarations, renames and handlers

**Rule.** Use a `declare` block to confine the scope of an object, a `renames`, or a `use` to the code that needs it, and to attach an exception handler to a small region so that the cause can be localised and execution can continue after recovery.

**Detect.** `assisted`. Objects declared at the top of a long subprogram but used only in one region; a `renames` far from its uses; a subprogram-level handler catching exceptions from one statement.

**Apply.** `assisted`. Moving a declaration into a block is safe when it is not used outside the region.

**Source.** Q&S95 §5.6.9, §5.7.2, §5.8.4.

---

## STM-18 Aggregates instead of component-wise assignment

**Rule.** Assign all components of a record with one aggregate (named association) rather than a sequence of component assignments, and build an actual parameter with an aggregate rather than through a temporary. Use positional association only when the components have a conventional order (`(X, Y)`). Adding a component to the type then makes the compiler flag every incomplete aggregate, which it cannot do for a list of assignments.

**Detect.** `auto`. Three or more consecutive `R\.\w+\s*:=` on the same record object; a temporary filled component-wise and then passed once.

**Apply.** `auto` when every component is assigned; `assisted` when only some are (Ada 2022 delta aggregates `(R with delta A => 1)` cover the partial case, see [14-modern-ada.md](14-modern-ada.md)).

**Source.** Q&S95 §5.6.10; §10.4.5.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| STM-01 | Name nested loops and loops with exits; name the exits | auto |
| STM-02 | Name nested blocks | auto |
| STM-03 | Name on every `end` | auto |
| STM-04 | `'First`/`'Last`/`'Range`/subtype name instead of literals | auto/assisted |
| STM-05 | Parentheses for clarity and forced order | auto |
| STM-06 | Positive logic | assisted |
| STM-07 | Short-circuit when the first operand guards the second | auto |
| STM-08 | No `=` on reals | assisted |
| STM-09 | Nesting at most five deep; `elsif` | assisted |
| STM-10 | Slices, not copy loops | auto |
| STM-11 | Complete explicit `case` over `if`/`elsif`; no literal ranges, minimal `others` | auto/assisted |
| STM-12 | `for` > `while` > plain loop with `exit`; no `exit` in `for`/`while` | assisted |
| STM-13 | `exit when` over `if ... then exit` | auto |
| STM-14 | Safety bounds on loops and recursion in critical code | advisory |
| STM-15 | No `goto` | assisted |
| STM-16 | Few, visible returns; `return Condition` | auto/assisted |
| STM-17 | Blocks to localise declarations, renames and handlers | assisted |
| STM-18 | Aggregates over component-wise assignment | auto |
