# 14. Modern Ada: Ada 2005, 2012 and 2022

Rules `MOD-*`. Sources: Rationale for Ada 2005 (Barnes; `corpus/2005/rationale/Rat-N-M.html`), Rationale for Ada 2012 (Barnes; `corpus/2012/rationale/Rat12-N-M.html`), Rationale for Ada 2012 Technical Corrigendum 1 (`RCorr-N.html`), Overview of Ada 2022 (Cousins; `corpus/2022/rationale/Ada2022-Overview.pdf`), Ada 2022 Annotated Reference Manual (`corpus/2022/language reference manual (annotated)/AA-*.html`).

The Ada 95 Quality and Style guide predates these revisions. Every rule in this file says "replace idiom X, which the Ada 95 guide recommended or tolerated, with feature Y", gives the revision that introduced Y, and quotes the rationale's own argument for why Y is better. Each rule is gated on the project's language version.

**Determine the language version before applying anything here.** Look, in order, at: the project file switches (`-gnat95`, `-gnat2005`, `-gnat2012`, `-gnat2022`, `-gnatX`), `pragma Ada_2012`/`Ada_2022` in sources or `gnat.adc`, the Alire manifest, and the compiler's documented default. When no evidence exists, assume Ada 2012 and say so in the report. Never apply a rule from a later revision than the project compiles with; report it as "available if the project moves to Ada YYYY".

Apply levels are as elsewhere: `auto` is an exact transformation that the compiler then verifies; `assisted` needs the reviewer to confirm the intent; `advisory` is a design change proposed, not made.

---

## Part A: Ada 2005

## MOD-01 `raise ... with "message"`

**Rule (Ada 2005).** Write `raise Some_Error with "message";` instead of `Ada.Exceptions.Raise_Exception (Some_Error'Identity, "message");`.

**Why.** The rationale calls the Ada 95 form "a lot neater" to replace, and the two are identical in effect. Note `raise E;` carries an implementation-defined message and `raise E with "";` a null one; keep that distinction when converting.

**Detect.** `auto`. `Raise_Exception\s*\(`.

**Apply.** `auto`. Also delete the `with Ada.Exceptions;` if nothing else uses it.

**Source.** Rat 2005 §6.2, §1.3.5.

---

## MOD-02 Overriding indicators on every primitive of a type extension

**Rule (Ada 2005).** Write `overriding` before every subprogram that overrides an inherited primitive and `not overriding` before every new primitive of a tagged type extension (and of a derived untagged type where the project agrees). The compiler then rejects a misspelt `Finalise` or a wrong profile that would silently add an overload instead of overriding. Put the indicator on its own line above the specification, or at the start of the same line; follow the project's existing choice. Do not write `overriding` in a partial view when the parent's operation is visible only in the full view: "the general rule is not to lie".

**Why.** "Careless profile errors lead to errors which are awkward to detect"; indicators "prevent errors during maintenance", because adding a parameter to the root operation then flags every derived type. "It is hoped that programmers will use overriding indicators freely."

**Detect.** `auto`. `procedure|function` declared after `type \w+ is new .* with` in the same spec, with no indicator; `Initialize`, `Adjust`, `Finalize` for controlled types without `overriding`.

**Apply.** `auto`. Insert the indicator that makes the declaration compile; a compile error after inserting `overriding` means the operation was not overriding, which is exactly the bug the rule exists to find. Report those separately.

**Example.**
```ada
overriding
procedure Finalize (Obj : in out T);          --  compile error if misspelt
not overriding
procedure Op (Obj : in out NT; Data : in String);
```

**Source.** Rat 2005 §2.7, §1.3.1.

---

## MOD-03 `limited with` for mutually dependent types; `private with` for private-part-only dependencies

**Rule (Ada 2005).** Two abstractions whose types refer to each other no longer need one "giant" package or a shared `*_Types` package of forward declarations: give each its own package and put `limited with Other;` on one of them (parameters of the incomplete view must be tagged, or be access parameters). When a `with` on a spec is needed only in the private part, write `private with`; it tells reviewers and tools that the visible part cannot pass on entities of that unit, and lets a public child's private part see a private sibling.

**Why.** "For many programmers the solution of the problem of mutually dependent types will be the single most important improvement introduced in Ada 2005"; the Ada 95 restriction "gives rise to huge cumbersome packages". And "if we have a situation where a private with clause is all that is needed then we should use it rather than a normal with clause".

**Detect.** `auto`. For each `with X;` on a `.ads`, no reference to `X` before the `private` keyword: candidate for `private with`. `assisted`: packages whose comment says they exist to break a dependency cycle; `Types`/`Common` packages full of `type T;`/`type T_Ptr is access T;` pairs ([STR-08](04-structure.md)).

**Apply.** `auto` for `private with` (compiler-checked; `use` clauses for that unit must move inside the private part). `advisory` for splitting packages with `limited with`.

**Source.** Rat 2005 §4.2, §4.3, §1.3.3.

---

## MOD-04 Interfaces for multiple inheritance of contracts; null procedures

**Rule (Ada 2005).** Replace a "signature" type (`type Printable is abstract tagged null record;` with all-abstract operations) that a type cannot also descend from, and the Ada 95 mixin-generic or access-discriminant techniques of [OOP-11](12-object-oriented.md) when their purpose was interface inheritance, with `type Printable is interface;` and `type T is new Parent and Printable with ...`. Use `synchronized`, `task` and `protected` interfaces to dispatch to protected or task operations. Give trivially empty operations as `is null` instead of a body containing `null;`.

**Why.** Multiple inheritance "is about merging contracts to be satisfied rather than merging algorithms or state". A synchronized interface guarantees any implementation "will be task safe because it can only be implemented by a task or protected type". Keep mixins for inheriting *implementation*.

**Detect.** `auto`. `is abstract tagged null record;` with only abstract operations; `procedure \w+ \(.*\) is\s*begin\s*null;\s*end`. `assisted`: wrapper tagged types around protected objects (`record PO : Protected_Type; end record`).

**Apply.** `assisted` for interface conversion (clients deriving from the signature type must change to `new X and Printable`); `auto` for null procedures.

**Source.** Rat 2005 §2.4, §5.3, §1.3.1, §1.3.4.

---

## MOD-05 Prefixed notation for tagged types

**Rule (Ada 2005).** For primitive and class-wide operations of tagged types whose first parameter is the object, write `Obj.Op (...)` instead of `Pkg.Op (Obj, ...)`, and `Ptr.Op` instead of `Op (Ptr.all)`. Keep the functional form for symmetric operations (`Sets.Union (A, B)`, not `A.Union (B)`) and for tag-indeterminate functions (`Empty`, `Unit (N)`). Choose one convention per project; mixed use is the finding.

**Why.** "Perhaps the most important advantage is ease of maintenance from not having to mention the package containing the declaration of the operation"; it also lets a component become a function without changing clients.

**Detect.** `auto`. `\w+\.\w+\s*\(\s*\w+\s*[,)]` where the package is the one declaring the first actual's tagged type; `\(\s*\w+\.all\s*\)`.

**Apply.** `auto` where the project already uses prefixed notation elsewhere; otherwise `advisory`, since it is a project-wide style choice.

**Source.** Rat 2005 §2.3, §1.3.1.

---

## MOD-06 Null exclusions, `access constant`, and anonymous access-to-subprogram parameters

**Rule (Ada 2005).** Where a parameter, component or named access type is documented as never null, or where the body begins with `if P = null then raise ...`, write `not null access T` (or `type A is not null access ...`) and delete the guard. Where an access parameter is only read through, write `access constant T`. Where a generic exists only to pass a callback (`generic with function F (X : Float) return Float;` instantiated once per use), or where a named access-to-subprogram type forces callbacks to be library-level, take `access function (X : Float) return Float` as a parameter so nested subprograms can be passed. Keep generics where the parameter is an operator (`"<"`), which cannot be passed as an access value. In Ada 2005 and later, `access T` parameters *admit* null (the check moves to the dereference), so Ada 95 code relying on the implicit check must add `not null` when upgraded.

**Why.** "The advantage of using a null exclusion is that when we come to do a dereference ... no check is required"; "in newly written programs, we should be encouraged to write not null explicitly". On closures: "the continued use of generics when other languages forty years ago had included a more natural mechanism was tiresome".

**Detect.** `auto`. `if \w+ = null then\s*raise`; `-- must not be null`; `access T` parameters whose bodies never assign through them; generics with a single `with function`/`with procedure` formal instantiated inside subprograms.

**Apply.** `assisted`. Adding `not null` moves the failure from a `raise` in the body to `Constraint_Error` at the call site, which is a behaviour change callers may depend on.

**Source.** Rat 2005 §3.2, §3.4, §1.3.2.

---

## MOD-07 `Ada.Containers` instead of hand-rolled structures

**Rule (Ada 2005).** Replace project-written linked lists (`type Cell; type Cell_Ptr is access Cell; ... Next : Cell_Ptr`), growable arrays with a separate count and copy-on-resize, hash tables, binary trees and local sort procedures with `Ada.Containers.Vectors`, `Doubly_Linked_Lists`, `Hashed_Maps`/`Ordered_Maps`, `Hashed_Sets`/`Ordered_Sets`, the `Indefinite_*` forms and `Generic_Array_Sort`. Use `Natural`/`Positive`, not `Integer`, as a vector index; use the definite forms unless the element is indefinite; prefer `Append (V1, V2)` to `V1 := V1 & V2`; remember containers are not task-safe and that modifying a container from inside `Query_Element`/`Update_Element`/`Iterate` callbacks raises `Program_Error`.

**Why.** Containers "bring the best in standard data structure manipulation to the fingers of every Ada programmer"; complexity bounds "avoid junk algorithms such as bubble sort"; cursors "identify both element and container so that we don't accidentally meddle with an element in the wrong container".

**Detect.** `auto` candidates: `Next\s*:\s*\w+_Ptr` list records with `Unchecked_Deallocation`; `Count : Natural` beside a fixed array in the same record; subprograms named `Sort`, `Insert_Sorted`, `Find_Node`, `Hash_Lookup`.

**Apply.** `advisory`. The structure's clients change; propose per structure with the matching container and the operation mapping.

**Source.** Rat 2005 §8.1, §8.2, §1.3.6.

---

## MOD-08 Limited aggregates, the `<>` box, and extended return

**Rule (Ada 2005).** Objects of limited types may be initialised with an aggregate and declared `constant`; write `X : constant T := (A => 10, B => True);` instead of declaring `X : T;` and assigning components. Use `<>` for components that cannot be given a value (protected components, "don't care" array elements) so the aggregate keeps full coverage checking. Build function results of limited or large types in place with `return R : T do ... end return;` instead of a local `Result` variable copied back; functions that returned an existing limited object "by reference" in Ada 95 must return `access T` instead.

**Why.** Component-by-component assignment "is annoying but it is prone to errors as well ... one of the advantages of aggregates is that we have to supply all the components"; with extended return "no copying is therefore ever performed".

**Detect.** `auto`. `\w+ : T;` of a limited type followed by consecutive `X.A := ...` lines ([STM-18](06-statements.md)); `Result : T;` ... `return Result;` in functions returning limited, discriminated or large types.

**Apply.** `auto` for the aggregate when all components are assigned; `assisted` for extended return.

**Source.** Rat 2005 §4.4, §4.5, §9.1.1.

---

## MOD-09 Concurrency idioms: discriminants, timing events, termination handlers, Ravenscar

**Rule (Ada 2005).** Parameterise tasks and protected objects with discriminants, not an initial `accept Initialize` ([CON-05](09-concurrency.md)). Replace a task whose only job is `accept Start; delay until T; Do_Something;` with `Ada.Real_Time.Timing_Events` and a protected handler. Replace the identical `when others => Log (...)` handler copied into every task body ([CON-19](09-concurrency.md)) with one `Ada.Task_Termination` fallback handler set during elaboration (keeping per-task handlers only where a task must recover rather than die). Replace a hand-written list of `pragma Restrictions`, `Task_Dispatching_Policy` and `Locking_Policy` with `pragma Profile (Ravenscar)` where the restrictions fit (Ada 2022 adds the less restrictive `Jorvik`). Use `PO'Priority := ...` inside a protected operation instead of inflating a ceiling.

**Why.** Timing events are "very low level and [do] not involve Ada tasks at all"; in Ada 95 an unhandled task exception "is propagated into thin air and just vanishes"; a profile "is equivalent to a set of configuration pragmas" and includes `Detect_Blocking`.

**Detect.** `auto`. Task bodies starting with `accept (Initialize|Start|Init)`; task bodies consisting of `delay until` then one call; identical `when others` handlers across task bodies; `pragma Restrictions (No_Abort_Statements, No_Task_Hierarchy, ...` lists.

**Apply.** `assisted`.

**Source.** Rat 2005 §5.2, §5.4, §5.5, §5.6, §1.3.4.

---

## MOD-10 `pragma Assert`, `No_Return`, `Unchecked_Union`, portability restrictions

**Rule (Ada 2005).** Write internal sanity checks as `pragma Assert (Cond, "message")` under `Assertion_Policy`, not `if not Cond then raise Program_Error; end if;` (never rely on side effects in the asserted expression, which is skipped under `Ignore`). Mark error-reporting procedures `pragma No_Return` (aspect in 2012) and delete the dummy `return`/`raise` after their calls. Map C unions with `pragma Unchecked_Union` instead of address overlays or `Unchecked_Conversion`, and only for C interfacing. Add `pragma Restrictions (No_Dependence => ..., No_Implementation_Pragmas, No_Implementation_Attributes, No_Obsolescent_Features)` to enforce [POR-01](10-portability.md) and [POR-06](10-portability.md) mechanically.

**Detect.** `auto`. `if not .* then\s*raise Program_Error`; `Fatal_Error (...); (return|raise)`; `for \w+'Address use \w+'Address` with a comment mentioning union.

**Apply.** `assisted` (`auto` for the `No_Return` pragma plus dummy-statement removal, compiler-verified).

**Source.** Rat 2005 §6.4, §1.3.5.

---

## MOD-11 Ada 2005 standard library instead of bindings and hand-written utilities

**Rule (Ada 2005).** Use `Ada.Directories` (exists, size, modification time, copy, path composition, search) instead of `Interfaces.C` imports of `stat`/`opendir` or `Text_IO` copy loops; `Ada.Environment_Variables` instead of `getenv` imports; `Ada.Calendar.Formatting`, `.Time_Zones`, `.Arithmetic` instead of hand-written `Day_Of_Week`, `H:M:S` splitting and ISO-8601 formatting; `Ada.Characters.Conversions` instead of the (now obsolescent) conversions in `Characters.Handling`; `Index (S, Pattern, From => I)` instead of slice-and-search; `Set_Unbounded_String` instead of `U := To_Unbounded_String (...)` in hot paths; `Ada.Text_IO.Unbounded_IO`/`Bounded_IO` and the function form of `Get_Line` instead of fixed buffers and `Get_Line (Buf, Last)` followed by `Buf (1 .. Last)`; `Generic_Real_Arrays`/`Generic_Complex_Arrays` instead of hand-written matrix loops; `'Stream_Size` to make stream files portable; `S'Mod (Offset)` instead of sign-tested signed-to-modular arithmetic.

**Why.** `Day_Of_Week` "is a nasty calculation"; the procedure `Get_Line` on an exactly full buffer returns alternating lines and null strings, which the function form avoids "without bother"; `Set_Unbounded_String` "avoids assignment and thus calls of Adjust".

**Detect.** `auto`. `Import (C, .*, "(getenv|stat|opendir|readdir|fopen)")`; local `Day_Of_Week`, `Is_Leap_Year`, `Image` of times; `Get_Line (\w+, Last)`; `To_Unbounded_String (Slice (`; `Handling\.To_Wide_String`.

**Apply.** `assisted`; the `Get_Line` function form and `Index ... From` are `auto`.

**Source.** Rat 2005 §6.3, §7.3, §7.4, §7.5, §7.6, §7.8, §1.3.6.

---

## Part B: Ada 2012

## MOD-12 Aspect specifications instead of representation pragmas and attribute clauses

**Rule (Ada 2012).** Write `with Inline`, `with Import, Convention => C, External_Name => "..."`, `with Pack`, `with Unchecked_Union`, `with Priority => P`, `with No_Return`, `with Atomic`, `with Volatile`, `with Size => 8`, `with Component_Size => 1`, `with Alignment => 4`, `with Address => ...`, `with Storage_Pool => ...`, `with Discard_Names` on the entity instead of the separate pragma or attribute definition clause. For a subprogram with a separate specification, put the aspect on the specification only. Keep `pragma Pure`, `Preelaborate`, `Elaborate_Body` and `Preelaborable_Initialization` as pragmas (program-unit pragmas remain the norm; the last has no aspect form until Ada 2022). Record and enumeration representation clauses have no aspect form and stay as they are.

**Why.** Pragmas "look ugly and verbose" and "it has to be clarified to which procedure Push they apply"; "having introduced aspect specifications which are generally so much nicer than pragmas, it was decided to allow aspect specifications for all those situations where pragmas are used ... and then to make most of the pragmas obsolete". AARM22 Annex J.15 lists the pragmas now obsolescent.

**Detect.** `auto`. `pragma (Inline|Import|Export|Convention|Pack|Unchecked_Union|Priority|Interrupt_Priority|No_Return|Atomic|Volatile|Atomic_Components|Volatile_Components|Storage_Size|Attach_Handler|Interrupt_Handler|Relative_Deadline|Asynchronous|Discard_Names|CPU|Dispatching_Domain)\s*\(`; `for \w+'(Size|Component_Size|Alignment|Address|Storage_Pool|Small|Bit_Order|Read|Write|Input|Output) use`.

**Apply.** `auto`. Exact transformation; compile. The one semantic difference: an aspect's expression is evaluated at the freezing point, so forward references become legal, never illegal.

**Example.**
```ada
procedure Set_Click (P : in Response)
  with Import, Convention => C;
type Byte is range 0 .. 255
  with Size => 8;
```

**Source.** Rat 2012 §2.2, §1.3.1; AARM22 J.15.

---

## MOD-13 `Pre` and `Post` instead of guards and "requires" comments

**Rule (Ada 2012).** Move a precondition that the body checks and raises on (`if Is_Empty (S) then raise Stack_Error;`), or that a header comment states (`-- Requires: S not empty`), to `with Pre => not Is_Empty (S)` on the specification, and state the result with `Post => ...` using `'Old` and `'Result`. For dispatching operations use `Pre'Class`/`Post'Class`, which the compiler combines so that overriding weakens preconditions (or-ed, checked at the call) and strengthens postconditions (and-ed, checked in the body), the discipline of [OOP-02](12-object-oriented.md). Specific `Pre`/`Post` are not inherited. Keep in mind: contracts are governed by `Assertion_Policy` and may be `Ignore`d in production, so a check whose failure must always be reported stays a body check or uses a raise expression (`MOD-25`); `Assertion_Error` cannot be handled inside the body; `'Old` is illegal on potentially unevaluated subexpressions and on limited objects.

**Why.** "It is the specification that provides the essence of the contract between the caller of the procedure Pop and the writer of its body. The contract provided by a traditional Ada subprogram specification is rather weak."

**Detect.** `auto`. First statement of a body is `if ... then raise`; `pragma Assert` as the first or last statement; comments `-- Pre:`, `-- Requires:`, `-- Precondition`, `-- Post:`, `-- Ensures:`.

**Apply.** `assisted`. Confirm with the project whether assertions are enabled in production before removing a body check.

**Example.**
```ada
procedure Pop (S : in out Stack; X : out Item)
  with Pre  => not Is_Empty (S),
       Post => not Is_Full (S);
```

**Source.** Rat 2012 §2.3, §1.3.1; RCorr §2.

---

## MOD-14 Type invariants and default initial conditions instead of consistency-check calls

**Rule (Ada 2012; Ada 2022 for `Default_Initial_Condition`).** When every exported operation of a private type ends with `Check_Consistent (Obj)` or `pragma Assert (Is_Valid (Obj))`, or a comment states a relationship between components, declare `Type_Invariant => Is_Valid (T)` on the private type (or on the full type, hidden from clients). For tagged hierarchies use `Type_Invariant'Class` and do not inherit specific invariants ("it is good advice not to use inheritance with specific invariants"). State what a default-initialised object satisfies with `Default_Initial_Condition => Is_Empty (Set)` (Ada 2022) rather than a comment or an `Initialized : Boolean` flag. Invariants are checked only at the boundary of the abstraction; components reached through access values are not covered.

**Detect.** `auto`. The same `pragma Assert (\w+ (\w+))` or call at the end of several bodies in one package; comments "invariant"; `Initialized : Boolean := False` components.

**Apply.** `assisted`.

**Source.** Rat 2012 §2.4; RCorr §2; Ada 2022 Overview §3.10.

---

## MOD-15 Subtype predicates instead of value-set tests and reordered enumerations

**Rule (Ada 2012).** A non-contiguous set of values tested repeatedly (`if M = Dec or M = Jan or M = Feb`), a Boolean function that exists only to classify (`Is_Winter`), or an enumeration reordered so that a range works, becomes `subtype Winter is Month with Static_Predicate => Winter in Dec | Jan | Feb;`, then `case` statements over the subtypes get full coverage checking and `for W in Winter loop` works. Prefer the case-expression form of the predicate when the base type may grow (a missing literal then fails to compile). Use `Dynamic_Predicate` for non-static conditions (`Even mod 2 = 0`). Know the limits: predicates are not checked on component assignment; predicated subtypes cannot index arrays or be sliced; `'First`/`'Last`/`'Range` are illegal on them (use `'First_Valid`/`'Last_Valid`); only static predicates on static subtypes allow `for` loops and aggregates; failures raise `Assertion_Error` unless `Predicate_Failure` says otherwise (TC1).

**Why.** "The subtype mechanism is somewhat limited. We can only specify a contiguous range of values"; a case statement over predicated subtypes "is also maintenance free".

**Detect.** `auto`. `\b(\w+) = \w+ or \1 = \w+ or \1 = \w+`; `function Is_\w+ \(\w+ : T\) return Boolean` whose body is a membership or equality chain on literals.

**Apply.** `assisted`.

**Source.** Rat 2012 §2.5, §1.3.1; RCorr §2.

---

## MOD-16 `Default_Value` and `Default_Component_Value`

**Rule (Ada 2012).** Where every declaration of a scalar type carries `:= 0`/`:= False`/`:= Unknown`, or a scalar is wrapped in a one-component record purely to obtain a default, give the type `with Default_Value => Unknown` (ideally a value that *means* uninitialised) or `with Default_Component_Value => 0` for arrays. The predefined types cannot take defaults: "this is a good reason for declaring our own types". With TC1, a type with `Default_Value` makes its `out` parameters copy-in, so junk cannot appear in `out` scalars left unassigned on some path.

**Detect.** `auto`. The same initialiser on most declarations of one type across a project; `type \w+ is record X : Scalar := V; end record;` single-component records; `out` scalar parameters assigned on only some paths (GNAT `-gnatwv`).

**Apply.** `assisted`. Defaults are static; adding one to a type whose objects were deliberately left uninitialised for speed changes performance in object-heavy code.

**Source.** Rat 2012 §2.6; RCorr §3, §4.

---

## MOD-17 If and case expressions

**Rule (Ada 2012).** Replace `Boolean'Pos (C) * A + Boolean'Pos (not C) * B` with `(if C then A else B)`; replace an `if`/`else` whose branches differ in one operand (`P (A, B, D)` / `P (A, C, D)`) with one call containing an if expression; replace a `case` (or `if`/`elsif` chain) whose every branch assigns one target with a case expression, which restores full coverage checking. Always parenthesise; omit the `else` of a Boolean if expression only when `else True` is meant; do not add `when others` to a case expression, which "would lose the value of coverage checking". Untaken static branches are statically unevaluated, so `(if Count = 0 then 0.0 else Total / Count)` is safe.

**Why.** "The absence of conditional expressions is a pain. It leads to unnecessary duplication"; with an if-expression chain over an enumeration "we have lost the full coverage check which is such a valuable feature of case statements".

**Detect.** `auto`. `Boolean'Pos\s*\(`; `case .* is` where every `when` body is a single assignment to the same variable; `if`/`else` branches identical except for one argument.

**Apply.** `auto` for the case-statement-to-case-expression rewrite when each branch is exactly one assignment to the same object; `assisted` otherwise.

**Example.**
```ada
Hours := (case D is when Mon .. Thu => 8, when Fri => 6, when Sat | Sun => 0);
```

**Source.** Rat 2012 §3.2, §3.3, §1.3.2; RCorr §3.

---

## MOD-18 Quantified expressions instead of flag-and-exit search loops

**Rule (Ada 2012).** Replace `Found := False; for K in A'Range loop if A (K) = X then Found := True; exit; end if; end loop;` with `Found := (for some K in A'Range => A (K) = X);` and the all-elements form with `for all`; use `for all E of AA` over any number of dimensions. Evaluation stops as soon as the result is known. Over an empty range `for all` is True and `for some` is False (TC1). These are the natural language of `Post` conditions ("sorted" is `(for all K in A'First .. Index'Pred (A'Last) => A (K) <= A (Index'Succ (K)))`).

**Detect.** `auto`. `(Found|Done|Exists|All_\w+|Any_\w+)\s*:=\s*(False|True);` immediately followed by a loop that only sets the flag and exits.

**Apply.** `auto` when the loop body is exactly the flag assignment and `exit`; `assisted` when it does anything else.

**Source.** Rat 2012 §3.4, §1.3.2; RCorr §3.

---

## MOD-19 Expression functions for one-line functions

**Rule (Ada 2012).** A function whose body is a single `return Expr;` becomes `function F (...) return T is (Expr);`, usually `with Inline`, declared where its meaning is visible: in the visible part when the expression is public, or declared in the visible part and completed in the private part when it refers to a private type's components. A procedure whose body is `null;` becomes `is null`. Completions by expression function cannot carry aspects (put `Inline` on the declaration) and cannot take `Pre`/`Post` (TC1).

**Why.** "What we need is a localized and visible shorthand for a little function"; information hiding of the body "is usually a good thing but here it is a problem" when a `Pre` refers to the function.

**Detect.** `auto`. Function bodies whose statement list is exactly one `return`; procedure bodies whose statement list is exactly `null;`.

**Apply.** `auto`. The transformation is exact. Keep a separate declaration in the spec and move the expression function into the private part when it touches private components.

**Source.** Rat 2012 §3.5, §1.3.2; RCorr §3.

---

## MOD-20 Extended membership tests

**Rule (Ada 2012).** Write `M in Jan .. Apr | Sep .. Dec` for `M in Jan .. Apr or M in Sep .. Dec`; `N in 6 | 28 | 496` for an equality chain (when no subtype predicate is warranted, `MOD-15`); `Obj in Triangle | Circle` for `Obj'Tag = Triangle'Tag or ...`; and test `X in Ptr_Type` before an access conversion instead of handling `Constraint_Error`. Choices are evaluated in order and short-circuit. Membership tests can no longer be `case` choices, and `A in B and C` needs parentheses (TC1).

**Why.** "The restrictions are annoying"; a membership test before a conversion "is much neater than attempting the conversion and then handling Constraint_Error".

**Detect.** `auto`. `(\w+) in [^;]+ or \1 in`; `(\w+) = \w+ or \1 = \w+` ([STM-06](06-statements.md) reports the positive-logic side); `'Tag = .*'Tag or`; `when Constraint_Error` around a type conversion.

**Apply.** `auto` for the first three; `assisted` for the conversion case.

**Source.** Rat 2012 §3.6, §6.4; RCorr §3.

---

## MOD-21 `in out` parameters on functions instead of access parameters used only for update

**Rule (Ada 2012).** A function taking `X : access T` solely so that it can modify `X.all` (random generators, cursors, counters) takes `X : in out T` instead. Functions with `in out` parameters are subject to the anti-aliasing rules (`Var := F (Var)` and `P (Var, F (Var))` are illegal), but evaluation order of operands remains unspecified, so a side-effecting function used twice in one expression is still an order dependence ([STR-05](04-structure.md), [POR-07](10-portability.md)). Operators keep `in` parameters only. Where a function must return a reference into a parameter, declare it `aliased in out`.

**Why.** Ada 95 was "the worst of all possible worlds. A function can perform any side effects at all, provided they are not made visible to the user by appearing as parameters of mode in out!"; the old rule "encouraged programmers to use access parameters unnecessarily with increased runtime cost and mental obscurity".

**Detect.** `auto`. `function \w+ \(\w+ : access \w+` whose body assigns `.all` or components and never stores the access value.

**Apply.** `assisted`. Callers change from `F (X'Access)` to `F (X)`, and objects passed may no longer need to be `aliased`.

**Source.** Rat 2012 §4.2, §1.3.3, §9.1.1.

---

## MOD-22 `use all type`

**Rule (Ada 2012).** Where `use type P.T;` is accompanied by qualified enumeration literals and primitive calls (`P.Mix (P.Red, P.Green)`), or where a package `use P;` exists only to reach one type's literals and primitives, write `use all type P.T;`. It makes operators, primitive operations, enumeration literals and class-wide operations of ancestors visible, while the type name itself stays qualified. This resolves the tension in [STR-15](04-structure.md) between readability and the ban on package `use` clauses: "the maintenance problem of name clashes really only applies to use package clauses".

**Detect.** `auto`. `use type \w+\.\w+;` plus `\1\.\w+` literal references in the same unit; `use P;` where every reference to `P`'s contents is an operation or literal of one type.

**Apply.** `auto`; compile to confirm no new ambiguity.

**Source.** Rat 2012 §4.5, §1.3.3.

---

## MOD-23 Generalised iteration and user-defined indexing

**Rule (Ada 2012).** Replace cursor loops (`C := L.First; loop exit when C = No_Element; ... C := Next (C); end loop;`), `Iterate (Do_It'Access)` with a separate procedure, and `for I in A'Range loop` where `I` is used only to index `A`, with `for E of L loop` (element form; `E` is a variable unless the container is constant) or `for C in L.Iterate loop` (cursor form, needed when the cursor matters, when starting from a given cursor, or when deleting). Replace `Update_Element (L, C, Put_Q'Access)` and `Temp := Element (C); Temp.Q := X; Replace_Element (L, C, Temp);` with `L (C).Q := X;` (user-defined indexing and implicit dereference). Loops over multidimensional arrays become one `for EE of AA`. State the element subtype as a checked comment when helpful: `for E : Twin of The_List`.

**Why.** "The detail of giving the precise description of the iteration and the indexing is really a violation of abstraction by revealing unnecessary detail"; "the possibility of making various errors of detail is completely removed". On `Update_Element`: "the first is slow, the second is gruesome (well, they are both gruesome really)".

**Detect.** `auto`. `= No_Element` loop exits; `:= Next\s*\(`; `Iterate\s*\(.*'Access\)`; `Update_Element|Query_Element|Replace_Element`; `for (\w+) in (\w+)'Range loop` whose body references `\1` only as `\2 (\1)`.

**Apply.** `auto` for the array index loop and the plain cursor loop (exact transformations); `assisted` when the body uses the index for anything else or deletes while iterating.

**Example.**
```ada
for E of The_List loop
   if Is_Prime (E.P) then
      E.Q := E.Q + X;
   end if;
end loop;
```

**Source.** Rat 2012 §6.3, §8.3, §1.3.5; RCorr §6.

---

## MOD-24 Small Ada 2012 clean-ups: `Yield`, no `null` after pragmas and labels, aspects for `Atomic`/`Volatile`, `Default_Storage_Pool`

**Rule (Ada 2012).** `delay 0.0;` as a scheduling point becomes `Ada.Dispatching.Yield;` (bounded error inside protected operations, as before; `Yield_To_Higher` under non-preemptive dispatching). `pragma Assert (...); null;` and `<<Label>> null;` drop the `null`. `pragma Volatile (X)` becomes `with Volatile` and spin-wait flags should be `Atomic`, since Volatile now only guarantees ordering, not memory access. Many identical `'Storage_Pool use My_Pool` clauses collapse to one `pragma Default_Storage_Pool (My_Pool)`; groups of objects with a common lifetime go in a subpool released with `Unchecked_Deallocate_Subpool`; `No_Standard_Allocators_After_Elaboration` enforces a "no allocation after start-up" policy that was previously a comment. Prefer named access types where allocation occurs; anonymous access types cannot be freed.

**Detect.** `auto`. `delay 0\.0;`; `delay until Ada\.Real_Time\.Time_First;`; `pragma Assert[^;]*;\s*null;`; `<<\w+>>\s*null;`; `pragma (Volatile|Atomic)`; three or more identical `'Storage_Pool use` clauses.

**Apply.** `auto` for the first four; `assisted` for pools.

**Source.** Rat 2012 §5.2, §5.5, §6.2, §6.4, §1.3.4, §1.3.5.

---

## MOD-25 Technical Corrigendum 1: raise expressions, `Predicate_Failure`, `X'Image`, `Discard_Names` aspect, stream aspects

**Rule (Ada 2012 TC1).** Compute-or-raise `if` statements become `Z := (if X < Y then 1 elsif X > Y then -1 else raise Error);`; a precondition that must raise an application exception uses `Pre => Cond or else raise Stack_Error with "full"`. A predicated subtype that should raise a domain exception gets `Predicate_Failure => raise Status_Error with "..."` instead of a `raise` inside the predicate (which breaks membership tests). `T'Image (X)` and GNAT's `X'Img` become `X'Image`. `pragma Discard_Names (T)` becomes `with Discard_Names`; `for T'Write use P;` becomes `with Write => P` (and `Write'Class` for class-wide streaming).

**Detect.** `auto`. `'Img\b`; `\w+'Image\s*\(\s*\w+\s*\)` where the prefix is a subtype of the argument; `pragma Discard_Names`; `raise` inside a `Dynamic_Predicate`.

**Apply.** `auto` for `'Image`, `'Img`, `Discard_Names`, stream aspects; `assisted` for raise expressions.

**Source.** RCorr §2, §3, §6, §7; Rat 2012 §9.5.

---

## MOD-26 Ada 2012 library: bounded containers, holders, synchronised queues, trees, string and directory additions

**Rule (Ada 2012).** In code that forbids heap allocation, use the `Bounded_*` containers (fixed `Capacity`, `Pure`, same interface as the unbounded forms, `Assign`/`Copy` to transfer between capacities) rather than arrays plus counts. Hold a single indefinite value (`String`, `T'Class`) in an `Indefinite_Holders.Holder` rather than an `access String` with manual `Free`. Replace a hand-written protected bounded buffer with `Unbounded_Synchronized_Queues`/`Bounded_Synchronized_Queues` (and the priority variants). Use `Multiway_Trees` rather than nodes with child lists. Use `Ada.Containers.Generic_Sort` to sort a sub-range or any indexable structure. For strings, use `Equal_Case_Insensitive`, `Less_Case_Insensitive`, `Hash_Case_Insensitive` instead of `To_Lower (A) = To_Lower (B)`, which is wrong for some scripts, and `Ada.Strings.UTF_Encoding` instead of byte juggling. Use `Ada.Directories.Hierarchical_File_Names` instead of `"/"` comparisons, `Environment_Variables.Value (Name, Default)` instead of the racy `Exists` then `Value`, and `Ada.Locales` instead of reading `LANG`.

**Detect.** `auto`. `To_Lower\s*\([^)]*\)\s*=\s*To_Lower`; `Hash\s*\(To_Lower`; `access String` components with `Free`; `protected type .* entry Put .* entry Get` buffers; `Exists\s*\("(\w+)"\).*Value\s*\("\1"\)`; `= "/"`; `Value\s*\("LANG"\)`.

**Apply.** `auto` for case-insensitive comparison and `Value` with `Default`; `assisted`/`advisory` for the containers.

**Source.** Rat 2012 §7.2 to §7.6, §8.2, §8.4 to §8.8, §1.3.6; RCorr §7.

---

## Part C: Ada 2022

## MOD-27 Target name `@`

**Rule (Ada 2022).** When the left-hand side of an assignment reappears on the right (`A.B (I).C := A.B (I).C + 1;`, `X := Natural'Min (X, 1000);`), write `@` for the repeated occurrence. This is a project-level style decision: the overview records that it "has proved to be rather controversial with those who are used to Ada being verbose". Apply it only where the project has adopted it.

**Why.** "Having a lengthy name multiple times in the same statement and having to mentally determine whether occurrences were the same was not helping the readability of Ada code"; "there is much less chance of a typo".

**Detect.** `auto`. Assignment statements whose right-hand side contains the exact text of the left-hand side.

**Apply.** `auto` if adopted; otherwise `advisory`.

**Source.** Ada 2022 Overview §7.7 (AI12-0125-3); Rat 2012 §9.4.4.

---

## MOD-28 Reduction expressions, aggregate iterators and filters, container and delta aggregates

**Rule (Ada 2022).** An accumulator loop (`Sum := 0; for I in A'Range loop Sum := Sum + A (I); end loop;`) becomes `A'Reduce ("+", 0)` or `[for I in 1 .. 10 => I**2]'Reduce ("+", 0)`. A loop whose whole body is guarded by one `if` becomes `for I in R when Cond loop`. A declare-then-fill array initialisation becomes `(for I in Index => I * I)` (index-parameter form keeps coverage checking). A container built by `X := Empty; Include (X, 1); Include (X, 2);` becomes `X := [1, 2];`, and a map `["Jeff" => True, "Randy" => False]`. Three or more assignments to components of one record become a delta aggregate `R := (@ with delta A => @.A + 1, B => @.B + V);`, and a postcondition that lists every unchanged component becomes `Post => D = (D'Old with delta Day => 12)`. A delta aggregate deliberately has no completeness check; a discriminant cannot be changed.

**Why.** "It is quite tedious to initialise a container, one has to create it as an empty container and then add elements one at a time"; delta aggregates "make it clear that the intent is to update all of these components as a group ... [and] the delta aggregate does not allow setting the same component twice by mistake". Caution on `'Parallel_Reduce`: for simple operations "the overhead of managing the parallelism is likely to outweigh any performance benefit".

**Detect.** `auto`. `(Sum|Total|Count|Product)\s*:=\s*[01];` followed by an accumulating loop; `loop\s+if .* then` where the `if` spans the whole body; `(Include|Insert|Append)\s*\(\s*(\w+),` repeated right after `\2`'s declaration; consecutive assignments to components of one prefix ([STM-18](06-statements.md)).

**Apply.** `auto` for `'Reduce` of a pure accumulation and for container aggregates of literal values; `assisted` for the rest. Note the Ada 2022 incompatibility: `Vectors.Append` of a whole vector is renamed `Append_Vector` (likewise `Insert_Vector`, `Prepend_Vector`).

**Source.** Ada 2022 Overview §2.5, §3.4, §4.5, §4.6, §4.7, §7.4.

---

## MOD-29 Contract refinements: declare expressions, `Stable_Properties`, contracts on access-to-subprogram and formal subprograms, `No_Return` functions, formal type defaults

**Rule (Ada 2022).** A long `Post` that repeats a subexpression names it once with `(declare Old_Pos : constant Position := S.Cur'Old; begin ...)`. A private type most of whose operations leave `Is_Open` and `Mode` unchanged declares `with Stable_Properties => (Is_Open, Mode)` instead of `X = X'Old` conjuncts on every operation. Callback types and generic formal subprograms carry `Pre`/`Post` (`type Callback is access procedure (X : Integer) with Pre => X mod 2 = 0;`). An error-reporting *function* may be `with No_Return`, removing dummy returns. A generic formal type that is almost always the same actual takes a default: `type Item_Count is range <> or use Natural;`.

**Detect.** `auto`. The same `'Old` subexpression three or more times in one `Post`; identical `\w+ \(X\) = \w+ \(X\)'Old` conjuncts across a package's postconditions; `return <dummy>; -- unreachable` after `raise` in functions; instantiations repeating the same actual for one formal.

**Apply.** `assisted`.

**Source.** Ada 2022 Overview §3.5 to §3.11.

---

## MOD-30 `'Image` for every type, `'Enum_Rep`/`'Enum_Val`, `Object_Size`, user-defined literals, big numbers, static expression functions

**Rule (Ada 2022).** Delete hand-written `Image`/`To_String` functions that concatenate component images and use `X'Image` on records, arrays and containers (output uses the aggregate syntax; customise with the `Put_Image` aspect and `Ada.Strings.Text_Buffers`). Replace `Unchecked_Conversion` between an enumeration with a representation clause and an integer with `T'Enum_Rep (X)` / `T'Enum_Val (N)`; this removes the last sanctioned use of unchecked conversion in [POR-16](10-portability.md). Use `with Object_Size => 8` when the allocated size, not the minimum `'Size`, must be fixed ("Users have been after this since 1983!"). Give abstract numeric and string types `Integer_Literal`, `Real_Literal`, `String_Literal` aspects so clients write `X := 3;` and `S := "text";` instead of `To_Big_Integer ("3")` / `To_Unbounded_String ("text")`. Use `Ada.Numerics.Big_Numbers` instead of `Long_Long_Integer` overflow workarounds or GMP bindings. Mark expression functions `with Static` so they can be used in static contexts instead of duplicating the formula in named numbers.

**Detect.** `auto`. `function (Image|To_String) \(\w+ : \w+\) return String is` whose body concatenates `'Image` calls; `Unchecked_Conversion \(.*(Source|Target) => \w+\)` where one side is an enumeration with a representation clause; `To_Unbounded_String \("`; `Interfaces\.C.*gmp`.

**Apply.** `auto` for `'Enum_Rep`/`'Enum_Val`; `assisted` for the rest (`'Image` output format differs from hand-written formats, which may be parsed elsewhere).

**Source.** Ada 2022 Overview §7.1, §7.3, §7.5, §7.11, §7.13.

---

## MOD-31 Static analysability aspects: `Nonblocking`, `Global`; `Max_Entry_Queue_Length`; Jorvik; atomic operations

**Rule (Ada 2022).** Replace "-- must not block (called from a protected operation)" comments and sole reliance on the run-time `pragma Detect_Blocking` with `with Nonblocking` on subprograms, protected types or whole packages, which the compiler checks statically (keep `Detect_Blocking` for the deadlock cases it alone catches). Replace `--# global` comments and "side effects:" prose with `with Global => (in X; out Y; in out Z)`, `Global => null` for pure functions (this gives [STR-05](04-structure.md) a checkable form), and `No_Unspecified_Globals` where a project wants it enforced. State per-entry queue bounds with `with Max_Entry_Queue_Length => 1`. Use `pragma Profile (Jorvik)` for projects that "almost" fit Ravenscar but need relative delays, `Ada.Calendar`, several protected entries or expression barriers. Replace machine-code or protected-object spin locks and counters with `System.Atomic_Operations.Exchange`/`Test_And_Set`/`Integer_Arithmetic`.

**Why.** "Applying pragma Detect_Blocking causes a run-time check, but this has a performance overhead, whereas applying aspect Nonblocking ... causes a check at compile-time"; Global annotations make it "easier for static analysis tools to reason, and (especially when there are no side-effects) can be useful to the human reader as well".

**Detect.** `auto`. `--#\s*global`; `-- (must not|never) block`; `pragma Detect_Blocking` with no `Nonblocking`; `-- at most one caller`; `pragma Profile \(Ravenscar\)` with vendor relaxations; `System.Machine_Code` with `xchg`/`cas`.

**Apply.** `assisted`.

**Source.** Ada 2022 Overview §2.3, §2.4, §3.1, §3.2, §6.3, §6.5, §6.10.

---

## MOD-32 Parallel constructs and procedural iterators

**Rule (Ada 2022).** Where an array of worker tasks exists only to partition an index range (`Slaves : array (1 .. N) of Worker` with `Start`/`Done` entries), consider `parallel for I in A'Range loop`, `parallel do ... and ... end do`, `parallel for E of Container`, with chunk specifications for partial sums, and `'Parallel_Reduce`. Blocking inside a parallel construct is a bounded error and conflicting global side effects are rejected under the default `Conflict_Check_Policy`; implementations may still run the construct sequentially, and the overview expects these "may be the last ones to be implemented". A named procedure written only to be passed to `Ada.Environment_Variables.Iterate`, `Ada.Directories.Search` or a container's `Iterate` becomes the body of `for (Name, Value) of Ada.Environment_Variables.Iterate (<>) loop ... end loop;` (`exit` is allowed only when the iterator has `Allows_Exit`).

**Detect.** `auto`. `array (.*) of \w+_Task` with partition arithmetic in the task bodies; `procedure \w+ (.*String.*)` declared immediately before `Iterate (\w+'Access)` or `Search (.*'Access)`.

**Apply.** `advisory` for parallel constructs (check compiler support); `auto` for procedural iterators when the compiler supports them.

**Source.** Ada 2022 Overview §2.1, §2.2, §2.6, §4.4, §4.9.

---

## Part D: Incompatibilities to check when raising the language version

Each revision adds reserved words and tightens a few rules. Before switching a project's `-gnatYYYY` switch, grep for these; the list is drawn from the "incompatibilities" sections of the rationales.

| Revision | Check for | Consequence |
|----------|-----------|-------------|
| 2005 | identifiers `Interface`, `Overriding`, `Synchronized` | now reserved words |
| 2005 | functions returning an existing limited object (`return Slaves (K);`) | must return `access T`; callers change |
| 2005 | `access T` parameters relied on for a null check | now admit null; add `not null` |
| 2005 | `subtype S is Acc (Disc => V)` on access to types with defaulted discriminants | illegal |
| 2005 | user-defined `"*"`/`"/"` on fixed-point types used with `Duration` | resolution changes; convert explicitly |
| 2012 | identifier `Some` | reserved word |
| 2012 | membership test as a `case` choice | illegal |
| 2012 | `Var := F (Var)` where `F` now has `in out` parameters | rejected by aliasing rules |
| 2012 | overloads differing in named versus anonymous access result | may become ambiguous |
| 2012 | `pragma Controlled` | removed |
| 2012 | `use` of `Ada.Characters.Handling`, `Ada.Containers`, `Ada.Directories`, `Ada.Strings.*`, `Ada.Tags` | new identifiers may clash |
| 2022 | identifier `Parallel` | reserved word |
| 2022 | `Vectors.Append`/`Insert`/`Prepend` of a whole vector | renamed `Append_Vector` etc.; `Append` loses its `Count` default |
| 2022 | `Y : Positive renames Positive'(X)` with `X` of a wider subtype | illegal |
| 2022 | `'Access` of a non-atomic subcomponent of an atomic object | illegal |

**Source.** Rat 2005 §9.1.1; Rat 2012 §9.1.1; Ada 2022 Overview §2.1, §4.5, §6.11, §7.12.

---

## Checklist

| ID | Rule | Since | Apply |
|----|------|-------|-------|
| MOD-01 | `raise E with "msg"` | 2005 | auto |
| MOD-02 | Overriding indicators on every primitive of an extension | 2005 | auto |
| MOD-03 | `limited with` for cycles; `private with` for private-only use | 2005 | auto/advisory |
| MOD-04 | Interfaces for contract inheritance; `is null` | 2005 | assisted |
| MOD-05 | Prefixed notation, except symmetric operations | 2005 | advisory/auto |
| MOD-06 | `not null`, `access constant`, anonymous access-to-subprogram parameters | 2005 | assisted |
| MOD-07 | `Ada.Containers` instead of hand-rolled structures | 2005 | advisory |
| MOD-08 | Limited aggregates, `<>`, extended return | 2005 | auto/assisted |
| MOD-09 | Discriminants, timing events, termination handlers, profiles | 2005 | assisted |
| MOD-10 | `Assert`, `No_Return`, `Unchecked_Union`, portability restrictions | 2005 | assisted |
| MOD-11 | Ada 2005 library instead of bindings and hand-written utilities | 2005 | assisted |
| MOD-12 | Aspects instead of representation pragmas and attribute clauses | 2012 | auto |
| MOD-13 | `Pre`/`Post` instead of body guards and comments | 2012 | assisted |
| MOD-14 | `Type_Invariant`, `Default_Initial_Condition` | 2012/2022 | assisted |
| MOD-15 | Subtype predicates instead of value-set tests | 2012 | assisted |
| MOD-16 | `Default_Value`, `Default_Component_Value` | 2012 | assisted |
| MOD-17 | If and case expressions | 2012 | auto/assisted |
| MOD-18 | Quantified expressions | 2012 | auto/assisted |
| MOD-19 | Expression functions and `is null` completions | 2012 | auto |
| MOD-20 | Extended membership tests | 2012 | auto |
| MOD-21 | `in out` function parameters instead of access-for-update | 2012 | assisted |
| MOD-22 | `use all type` | 2012 | auto |
| MOD-23 | `for E of`, `for C in Iterate`, user-defined indexing | 2012 | auto/assisted |
| MOD-24 | `Yield`, no `null` after pragmas/labels, `Atomic`/`Volatile` aspects, pools | 2012 | auto/assisted |
| MOD-25 | Raise expressions, `Predicate_Failure`, `X'Image`, `Discard_Names`, stream aspects | 2012 TC1 | auto/assisted |
| MOD-26 | Ada 2012 library: bounded containers, holders, queues, trees, strings, directories | 2012 | auto/advisory |
| MOD-27 | Target name `@` | 2022 | advisory/auto |
| MOD-28 | `'Reduce`, filters, aggregate iterators, container and delta aggregates | 2022 | auto/assisted |
| MOD-29 | Declare expressions, `Stable_Properties`, contracts on access/formal subprograms, `No_Return` functions, formal defaults | 2022 | assisted |
| MOD-30 | `'Image` for all types, `'Enum_Rep`, `Object_Size`, user literals, big numbers, `Static` | 2022 | auto/assisted |
| MOD-31 | `Nonblocking`, `Global`, `Max_Entry_Queue_Length`, Jorvik, atomic operations | 2022 | assisted |
| MOD-32 | Parallel constructs; procedural iterators | 2022 | advisory/auto |
