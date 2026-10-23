# 4. Program Structure and Visibility

Rules `STR-*`. Source: Ada 95 Quality and Style (Q&S95) chapter 4 (§4.1 High-Level Structure, §4.2 Visibility); Q&S83 chapter 4. Exceptions as part of an interface (Q&S95 §4.3) are in [08-exceptions.md](08-exceptions.md).

These are design-level rules. Most are `advisory`: the reviewer reports them with a concrete suggestion, and the restructuring is done only with agreement, because moving declarations between units changes the public interface and the build. A few (context clauses, variables in specs) are mechanical enough to apply directly.

---

## STR-01 One unit per file; specification and body in separate files

**Rule.** Put each library unit specification in its own file, separate from its body. Put each subunit in its own file. Use one consistent file naming convention (GNAT's default: `unit_name.ads` / `unit_name.adb`, child units with `-` for the dot, subunits named `parent-subunit.adb`; the guide shows `text_io_integer_io.adb`). Avoid library-level subprograms other than the main program; if one exists and is named in a `with` clause, give it an explicit specification in its own file.

**Why.** Bodies change far more often than specifications. A body sharing a file with its spec forces recompilation of the spec and of every client, with a ripple effect. Separate files also allow several bodies for one spec (per-target, or a test spec that exports everything). Library-level subprograms have no package body in which to keep their state; embed them in a package instead.

**Detect.** `auto`. A `.adb` file containing `package Foo is` (spec) before `package body Foo is`; several library units in one file; `procedure X is` at library level in a file that is not the main program and has no matching `.ads`.

**Apply.** `assisted`. Splitting a file is safe but changes the build; confirm the build picks up the new file (GNAT project source dirs, naming scheme).

**Source.** Q&S95 §4.1.1.

---

## STR-02 Prefer child units to subunits and to nesting in bodies

**Rule.** Minimise `is separate` subunits; structure a subsystem with child library units instead. For supporting subprograms and data needed by the parent body and by child units that extend the parent, put them in a private child package whose spec declares them, and `with` that private child from the bodies that need it, rather than nesting in the parent body.

**Why.** A subunit sees everything declared in the parent body before it (more global data, larger ripple from changes) and encourages copying code into the subunit rather than sharing it. A private child can be extended without recompiling the parent spec or its clients; nesting in the parent body makes helpers unavailable to child units, forcing changes to the parent spec.

**Detect.** `auto`. `is separate;` occurrences; package bodies over roughly 1000 lines with many nested subprograms. (GNAT: `-gnatw` has no check; count with grep.)

**Apply.** `advisory`. Propose the split; converting a subunit to a child unit changes visibility (the subunit saw the parent body's declarations; the child sees the parent spec, including its private part).

**Source.** Q&S95 §4.1.1, §4.1.6, §4.2.3.

---

## STR-03 Configuration pragmas outside the source, or in one unit

**Rule.** Express configuration pragmas (`Task_Dispatching_Policy`, `Restrictions`, `Profile`, `Assertion_Policy`, `Suppress` used as a configuration pragma) through compiler options or a configuration file (`gnat.adc`, project file switches) rather than in source. Where they must be in source, keep them in one compilation unit per partition, preferably the main subprogram.

**Why.** They make partition-wide decisions. Embedded in a reusable component they break reuse; scattered across the source they are hard to find when the decision changes.

**Detect.** `auto`. `^\s*pragma\s+(Restrictions|Profile|Task_Dispatching_Policy|Locking_Policy|Queuing_Policy|Assertion_Policy|Normalize_Scalars|Detect_Blocking|Partition_Elaboration_Policy)` in files other than the main unit or `gnat.adc`.

**Apply.** `assisted`. Move to `gnat.adc` or the project file; verify the build still applies them.

**Source.** Q&S95 §4.1.2.

---

## STR-04 Subprograms do one thing

**Rule.** Use subprograms as abstractions, each performing a single action. A subprogram does not need more than one caller to justify its existence.

**Detect.** `assisted`. Bodies longer than a screen or two with several distinct phases separated by comments like `-- Now do X`; procedures named `Process_And_Save`, `Read_Validate_Store`.

**Apply.** `advisory`. Extract subprograms. Mention Q&S95 §10.7.1 if a performance objection comes up: the call overhead is rarely the bottleneck, and `pragma Inline` exists.

**Source.** Q&S95 §4.1.3.

---

## STR-05 Functions return a value and have no side effects

**Rule.** Use a function when the primary purpose is to deliver a single value. Minimise side effects (any change to a non-local variable, including changes made by what the function calls). A program whose result depends on the order in which function side effects occur is erroneous, because Ada does not define the evaluation order of operands and actual parameters. Prefer a parameterless function over a constant when the value need not be static, when it should be inherited by derived types (a function is a primitive operation; a constant is not), or when the value may change later.

**Why.** Side effects are hard to understand and maintain. `Next_Character` used twice in one call has no defined ordering.

**Detect.** `assisted`. Functions whose body assigns to a package-level variable or calls a procedure with an `out`/`in out` global; Ada 2012 functions with `in out` parameters used in expressions with other calls; two calls to the same side-effecting function in one expression or argument list.

**Apply.** `assisted`. Convert to a procedure with an `out` parameter, or restructure so each side-effecting call is its own statement. In Ada 2012 and later, annotate pure functions with `Global => null` where SPARK or the project supports it, see [14-modern-ada.md](14-modern-ada.md).

**Source.** Q&S95 §4.1.4; Q&S83 §4.1.4.

---

## STR-06 Packages encapsulate

**Rule.** Use packages for information hiding; for abstract data types (private types, tagged types); to model domain entities; to group related type and object declarations shared by several library units; and to encapsulate machine, device and operating-system dependencies, so that a different device or target means a different body behind the same spec. Put low-level implementation decisions behind subprograms within packages.

**Detect.** `assisted`. `System`, `Interfaces.C`, `pragma Import`, address clauses or `Unchecked_Conversion` appearing in many unrelated units rather than one interface package. Types declared with all components public where a private type would do.

**Apply.** `advisory`.

**Source.** Q&S95 §4.1.5; §7.1.5.

---

## STR-07 Child units for extension and subsystems

**Rule.** When a new unit logically extends an existing abstraction, make it a child of that unit; when it introduces an independent abstraction, make it a separate library unit. Build a subsystem as a hierarchy: the root package holds the types, subtypes and constants that clients and children need; public children present the parts clients may see; private children hold what clients must not see, including local declarations used only to implement the parent and debugging utilities. Put constructors in a child package even when they return access values. Use children, not nested packages, to present different views of one abstraction; nest package specs only to group operations or share implementation detail (as `Ada.Strings.Bounded.Generic_Bounded_Length` does).

**Why.** A child's private part and body see the parent's private type, so related abstractions can share a representation without one monolithic package, and a child can be added without recompiling the parent or its clients.

**Detect.** `assisted`. Very large package specs with distinct clusters of operations; packages named `Foo_Utils`, `Foo_Extras`, `Foo_Internal` that are not children of `Foo`; `Unchecked_Conversion` or `Unchecked_Access` used to reach a sibling's representation.

**Apply.** `advisory`.

**Source.** Q&S95 §4.1.6, §4.2.1, §4.2.2, §3.2.8, §9.4.1.

---

## STR-08 Cohesion

**Rule.** Each package serves a single purpose. Group by function and by need to share data, not by category of activity (`Initialisation_Routines`) or timing. Avoid catch-all packages (`Project_Definitions`, `Globals`, `Common`, `Utils`, `Misc`); a traditional library of unrelated routines should be split into cohesive packages (trigonometry, string handling, and so on). Restructure when the first structure proves wrong.

**Detect.** `auto` candidates: packages named `Globals`, `Global_Data`, `Common`, `Utils`, `Utilities`, `Misc`, `Definitions`, `Types` (a project-wide `Types` package is a classic catch-all; a per-subsystem one is fine), `Constants`. `assisted` judgment.

**Apply.** `advisory`.

**Source.** Q&S95 §4.1.7; §5.4.2.

---

## STR-09 No variables in package specifications

**Rule.** Do not declare variables in a package specification. Keep state in the body (or a private child) and provide access subprograms, or model the state as a protected object when tasks are involved.

**Why.** A spec-level variable can be modified by any unit with visibility and cannot be protected or abstracted by its package. Objects that must persist belong in bodies.

**Detect.** `auto`. In a `.ads` file, outside `private`, an object declaration without `constant`: `^\s*\w+(\s*,\s*\w+)*\s*:\s*(?!constant\b)(?!exception\b)[\w.]+.*;` at package level (not inside a record, not a parameter). Also check the private part: variables there are still shared with children.

**Apply.** `assisted`. Move the object to the body and add `function Get_X`/`procedure Set_X` (or a better-named abstraction). Every external reader and writer must be updated. The guide's noted exception is performance, which is rarely decisive; `pragma Inline` on the accessors answers it.

**Example.**
```ada
package Compilation_Status is
   type Line_Number is range 1 .. 2_500_000;
   function Source_Line_Number return Line_Number;   --  not: Current_Line : Natural;
end Compilation_Status;
```

**Source.** Q&S95 §4.1.8; §4.2.1.

---

## STR-10 Tasks and protected types for the right jobs

**Rule.** Use tasks to model asynchronous entities in the problem domain, to express concurrent algorithms for multiprocessors, and for cyclic or prioritised activities. Use protected types to control or synchronise access to data or devices and for passive synchronisation such as a resource monitor. Do not use a task where a protected object will do.

**Detect.** See [09-concurrency.md](09-concurrency.md).

**Apply.** `advisory`.

**Source.** Q&S95 §4.1.9, §4.1.10, §6.1.1, §6.1.2.

---

## STR-11 Minimal interfaces

**Rule.** A package spec contains only what its users need. Do not add operations merely because they are easy to build. Reconsider subprograms with many parameters (the boundaries are probably wrong), but do not reduce parameter counts by having the subprogram touch global data, which is an undocumented interface. Design interfaces after working out what the expected clients actually do, not what they might need.

**Detect.** `auto` candidates: declarations in a spec never referenced outside the package (`gnatfind`/`gnatxref` or grep across the project); subprograms with more than about 7 parameters. `assisted` judgment.

**Apply.** `assisted`. Removing an unused public declaration is safe once the whole-project search is clean. Moving it to the body or a private child is safe.

**Source.** Q&S95 §4.2.1; §8.3.1.

---

## STR-12 Minimal context clauses, placed as low as possible

**Rule.** `with` only the units directly needed. Put a `with` on the body when only the body needs it, never on the spec. Where only one subunit or child needs a unit, put the `with` on that unit. In Ada 2005 and later, use `private with` when a unit is needed only in the private part, and `limited with` to break mutual dependencies between specs (see [14-modern-ada.md](14-modern-ada.md)).

**Why.** Fewer clauses on the spec means less to read, fewer recompilations when the withed unit changes, and a visible statement of what the interface really depends on.

**Detect.** `auto`. For each `with X;` on a `.ads`, check that `X` is referenced in the spec; for each `with X;` anywhere, check `X.` or a `use X;` appears in the unit (GNAT warns: `-gnatwu` unused withs, `-gnatwr` redundant). 

**Apply.** `auto`. Remove unreferenced `with` clauses; move spec-only-in-body clauses to the body; convert spec clauses used only in the private part to `private with`. Recompile to confirm.

**Source.** Q&S95 §4.2.1, §4.2.3.

---

## STR-13 Restrict visibility; minimise nesting in subprograms and tasks

**Rule.** Give each program unit the smallest visibility that works: a private child first; failing that, nest inside the package body (not the spec). Avoid nesting units inside subprograms and task bodies except to guarantee the unit is called from nowhere else.

**Why.** A unit nested in a subprogram makes up-level references into its defining context and is unreusable; a unit hidden in a body cannot be called from an unintended place, so local changes stay local.

**Detect.** `auto` candidates: subprograms declared in a spec but referenced only from within the package; subprograms nested 2+ levels deep inside other subprograms.

**Apply.** `assisted`. Moving a public subprogram to the body is safe once no external reference exists.

**Source.** Q&S95 §4.2.3; Q&S83 §4.2.3.

---

## STR-14 Decide deliberately whether to hide tasks

**Rule.** Choose between declaring a task in a package spec and hiding it in the body behind subprograms. Hiding reduces clutter, lets the subprograms enforce call ordering and defensive timed or conditional calls, and lets entries be grouped for different classes of user. But hiding also conceals blocking behaviour: if a client must reason about deadlock, starvation or race conditions, either expose the task or document in the spec header that a call may block. Hiding also prevents clients from using conditional and timed entry calls unless the procedures provide for them. Where performance matters and no ordering must be enforced, rename the entries as subprograms rather than wrapping them.

**Detect.** `assisted`. Package specs with no mention of tasking whose bodies declare tasks; spec headers that do not say which calls block.

**Apply.** `advisory`; `assisted` for adding the header note.

**Source.** Q&S95 §4.2.4; §6.2.2.

---

## STR-15 Use clauses: minimal, localised, `use type` for operators

**Rule.** Avoid or minimise `use` clauses for packages; name entities with their package prefix. Where operators are needed, write `use type T;` (Ada 95) or `use all type T;` (Ada 2012, which also brings in primitive operations and enumeration literals) instead of a package `use` or operator renamings. A package `use` is reasonable for standard packages when no ambiguity results, and for enumeration literals. Whatever `use` clauses remain go in the narrowest scope: a body, a subprogram, or a block, not the spec's context clause. Consider a package `renames` to shorten a long prefix instead of a `use`.

**Why.** With `use` clauses, a reader cannot tell which package an identifier comes from, external dependencies are obscured, and the maintainer must resolve overloads by hand. `use type` gives the natural infix notation without those costs, and avoids the classic error of renaming `"+"` as `"-"`.

**Detect.** `auto`. `^\s*use\s+[\w.]+;` in a `.ads` context clause; `use` of a package when the only entities used from it are operators (replace by `use type`); `function "\+" (...) renames` operator renamings.

**Apply.** `assisted`. Removing a package `use` requires qualifying every reference, which a compiler will enumerate; moving it from spec to body is safe when the spec compiles without it; replacing operator renamings by `use type` is safe. Tools exist that convert between the two forms automatically.

**Example.**
```ada
with Rational_Numbers;
with Rational_Numbers.IO;
procedure Demo_Rationals is
   package R_IO renames Rational_Numbers.IO;
   use type Rational_Numbers.Rational;
   use R_IO;
   X, Y : Rational_Numbers.Rational;
begin
   R_IO.Put (X / Y);
```

**Source.** Q&S95 §5.7.1; Q&S83 §5.7.1; ARM §J.1 (do not use the Ada 83 library-level renamings such as `Text_IO` in new code).

---

## STR-16 Renames: narrow scope, real purposes

**Rule.** Limit a renaming declaration to the smallest scope that needs it, and use renaming for: shortening an unwieldy fully qualified name used repeatedly in one region; providing a subprogram body that merely calls another subprogram (`procedure Disk_Write (...) renames System_Specific.Device_Drivers.Transmit;`, no pass-through body); giving a reusable component's badly named parts your project's nomenclature through an interface package; and project-wide package abbreviations from a standard list. Rename for visibility instead of a `use` clause, except for operators (`use type`). Do not rename a type; declare a subtype. Do not rename every package to an abbreviation, which defeats meaningful package names.

**Detect.** `auto`. `renames` at package-spec level of something used only in one subprogram; subprogram bodies whose only statement is a call forwarding all parameters; `subtype X is Y;` used as a type rename is fine, `type X is new Y;` used as a rename is not.

**Apply.** `assisted`. Moving a renaming into a block is safe; converting a pass-through body to a renaming is safe when the profiles match exactly.

**Source.** Q&S95 §5.7.2; §3.1.4; Ada 95 Rationale §II.12.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| STR-01 | Spec and body in separate files; consistent file naming; no stray library subprograms | assisted |
| STR-02 | Child units, not subunits or body nesting, to structure a subsystem | advisory |
| STR-03 | Configuration pragmas via switches or one unit | assisted |
| STR-04 | One action per subprogram | advisory |
| STR-05 | Functions: one value, no side effects; parameterless function vs constant | assisted |
| STR-06 | Packages hide information and encapsulate dependencies | advisory |
| STR-07 | Child units for extension, subsystems, constructors and views | advisory |
| STR-08 | Cohesive packages; no catch-alls | advisory |
| STR-09 | No variables in package specs | assisted |
| STR-10 | Tasks for active entities, protected types for shared data | advisory |
| STR-11 | Minimal interfaces; few parameters without hidden globals | assisted |
| STR-12 | Minimal `with` clauses, as low as possible; `private with` | auto |
| STR-13 | Smallest visibility; avoid nesting in subprograms and tasks | assisted |
| STR-14 | Decide and document whether tasks are hidden | advisory |
| STR-15 | Minimal, localised `use`; `use type` for operators | assisted |
| STR-16 | Narrow-scope renames for real purposes; no pass-through bodies | assisted |
