# 15. Usage Advice from the Language Rationales

Rules `RAT-*`. Sources: Rationale for the Design of the Ada Programming Language (Ada 83; `corpus/1983/rationale/ratl-NN-MM.html`), Ada 95 Rationale (`corpus/1995/rationale/rat95-pN-M.html`). Advice from the 2005, 2012 and 2022 rationales is in [14-modern-ada.md](14-modern-ada.md).

The rationales are design documents, not style guides, but their authors repeatedly say how a feature *should* be used. This file keeps the advice that the Quality and Style guide does not already state, or states less sharply. Each entry cites the rationale section so the reviewer can quote the designers' own argument.

---

## RAT-01 Every library unit is `Pure`, `Preelaborate` or `Elaborate_Body`, in that order of preference

**Rule.** "It is good advice to give all library units one of the pragmas Pure, Preelaborate, or Elaborate_Body in that order of preference, wherever possible." Use `Elaborate_All` (transitive) rather than `Elaborate` when a unit calls another during elaboration. A library package whose body exists only to run initialisation code must carry `Elaborate_Body`, otherwise the body is illegal (Ada 95) and in Ada 83 "could get inadvertently lost without any warning".

**Why.** Categorised units let the compiler eliminate access-before-elaboration checks and give the binder a determined order; `Pure` units can be shared between partitions and their calls omitted when results are unused.

**Detect.** `auto`. Library packages with no `Pure`/`Preelaborate`/`Elaborate_Body`; `pragma Elaborate (` (non-transitive).

**Apply.** `auto`. Try `Pure`, then `Preelaborate`, then `Elaborate_Body`; the compiler rejects the ones that do not apply. Replace `Elaborate` with `Elaborate_All` and rebuild; a new elaboration cycle means the dependency was real and must be designed away.

**Source.** Ada 95 Rationale §10.3, §10.4; [REU-17](11-reusability.md), [PRF-07](13-performance.md).

---

## RAT-02 Never compare tags to select behaviour; use membership tests or dispatching

**Rule.** Inside a primitive, `if A'Tag = High_Alert'Tag then` "is quite inappropriate since it has reintroduced the rigid nature of variant programming and could not specifically recognize an alert which is a later extension". Where a class-wide test is needed, write `AC in Medium_Alert'Class`; better still, give each type its own overriding and redispatch with a view conversion `Display (Medium_Alert'Class (MA), Console)`. View an object class-wide with a renaming (`AC : Alert'Class renames Alert'Class (A);`), not a copy. Remember the "golden rules": the tag of an object never changes; conversion is never away from the root; conversion never changes the tag.

**Detect.** `auto`. `'Tag\s*=`; `Ada.Tags.Expanded_Name` comparisons; `case` on `'Tag`.

**Apply.** `assisted`. Membership test is a drop-in replacement; moving the branch into overridings is a refactoring ([TYP-09](05-types.md)).

**Source.** Ada 95 Rationale §4.3, §4.5.

---

## RAT-03 Resource handles as `Limited_Controlled` objects; semaphores are the `goto` of tasking

**Rule.** Pair operations that must always be matched (lock/unlock, open/close, mark/release) by a local `Limited_Controlled` handle whose `Initialize` acquires and `Finalize` releases, so release happens on every exit "exception, exit, return, abort, asynchronous transfer of control". A user-released lock in a `when others =>` handler "often constitutes abstraction inversion since it leaves the responsibility for releasing the semaphore in the hands of the user". "Semaphores (like the goto statement) are very prone to misuse and should be avoided where possible"; use a protected object.

**Detect.** `auto`. `Lock (`/`Acquire (` with a matching `Unlock`/`Release` in the same body (and in a `when others` handler); `Open (` without `Close` on an exception path; `Suspension_Object` used as a mutex.

**Apply.** `assisted`. Introduce the handle type once, then replace each pair.

**Example.**
```ada
type Handle (Resource : access Some_Thing) is new Ada.Finalization.Limited_Controlled with null record;
overriding procedure Initialize (H : in out Handle);   --  Lock (H.Resource.all)
overriding procedure Finalize   (H : in out Handle);   --  Unlock (H.Resource.all)
...
declare
   Guard : Handle (Shared'Access);
begin
   ...   --  unlocked however this block is left
end;
```

**Source.** Ada 95 Rationale §7.4, §9.1; Ada 83 Rationale §14.3 ("last wishes").

---

## RAT-04 `'Valid`, not a membership test, for data from outside the type system

**Rule.** To check a scalar that may be uninitialised, unchecked-converted, read from another language or from a file, use `X'Valid`, not `X in 1 .. 10`: a membership test "may be optimized away" since the compiler may assume the value is in range, and "might not work as desired for enumeration types with 'holes'". Read external records into the constrained record type first and then test each scalar component. In high-integrity code use `pragma Normalize_Scalars` so uninitialised scalars hold invalid values, but never together with `No_Exceptions`; do not rely on a compiler that defaults scalars to in-range values, which "would mask errors".

**Detect.** `auto`. `in \w+'Range` or `in L .. H` applied to results of `Unchecked_Conversion`, `Read`, imported functions, or objects with address clauses.

**Apply.** `auto`. Replace the test by `'Valid` (and keep the range test only if the subtype is narrower than the type).

**Source.** Ada 95 Rationale §13.3, §H.1; [TYP-15](05-types.md).

---

## RAT-05 Time: `delay until`, `Ada.Real_Time`, `Milliseconds (N)`, named durations

**Rule.** Periodic activity is `delay until Next; ...; Next := Next + Period;` with `Ada.Real_Time` when the activity must not be disturbed by clock changes: `Calendar.Clock` "cannot be relied upon for measurement of physical time in real-time applications". Never `delay Next - Clock` (preemption between the two makes the delay too long). Build spans with `Milliseconds (5)` rather than `5 * One_Millisecond`, "to avoid cumulative conversion error". Express durations with named constants (`Minutes : constant := 60.0; delay 6 * Minutes;`).

**Detect.** `auto`. `delay\s+\w+\s*-\s*Clock`; `Ada.Calendar.Clock` in task bodies with periods; numeric literals in `delay` statements; `\d+\s*\*\s*One_Millisecond`.

**Apply.** `auto` for `delay until` and `Milliseconds`; `assisted` for switching clocks.

**Source.** Ada 95 Rationale §II.10, §9.3, §D.8; Ada 83 Rationale §13.2; [CON-09](09-concurrency.md).

---

## RAT-06 I/O idioms: `Current_Error` for diagnostics, non-generic `Integer_Text_IO`, save and restore default files

**Rule.** Send diagnostics to `Current_Error` (`Set_Error`) so real output and messages separate "in a portable manner". For simple numeric I/O use the non-generic `Ada.Integer_Text_IO`, `Ada.Float_Text_IO` (no instantiation, standard names) unless a formal package actual is needed. A widely used subprogram that changes the default output file saves it with `Old : constant File_Access := Current_Output;` and restores it; `File_Access` is access-to-constant so the saved reference cannot be passed to `Close`. Instantiate `Float_IO`/`Integer_IO` for the application's own numeric types rather than converting to `Float` for printing, which "introduces portability problems because the default format will depend upon the properties of the predefined type".

**Detect.** `auto`. `Put_Line \("Error`; `Set_Output (` without a matching restore; `Put \(Float \(`; `package \w+_IO is new Ada.Text_IO.Integer_IO \(Integer\)`.

**Apply.** `auto` for the non-generic packages and `Standard_Error`; `assisted` otherwise.

**Source.** Ada 95 Rationale §A.4; Ada 83 Rationale §16.3, §16.5; [POR-17](10-portability.md).

---

## RAT-07 Choose the string kind by storage behaviour

**Rule.** `String` when the length never changes; `Bounded_String` when a maximum is known at creation (provide "a small number of instantiations with sufficient maximum size, for example max lengths of 20 and 80"); `Unbounded_String` otherwise, embedded as a component when a richer structure is needed. Use the package's `"="` for bounded strings (predefined equality would compare junk past the logical length). The functional string operations "typically involve extra copying"; the procedural forms "can lead to a heavy-looking style": choose by need ([PRF-06](13-performance.md)). With a case-folding mapping such as `Lower_Case_Map`, make sure the pattern contains only mapped characters ("GERBIL" never matches).

**Detect.** `auto`. Many `Generic_Bounded_Length` instantiations with ad hoc maxima; `Unbounded_String` for fixed-format fields; `String (1 .. N)` buffers with a separate length variable (candidate for bounded).

**Apply.** `assisted`.

**Source.** Ada 95 Rationale §A.2.

---

## RAT-08 Money and other decimal quantities in decimal fixed point, never floating point

**Rule.** "Floating point is unacceptable because of the inherent inexactness of representing decimal quantities." Declare `type Money is delta 0.01 digits 18;`, round explicitly with `S'Round` (conversion truncates), and constrain precision with `digits` subtypes to save space. For external storage use `Interfaces.COBOL.Decimal_Conversions.To_Display`.

**Detect.** `auto`. `: Float` / `Long_Float` objects or components named `Price`, `Amount`, `Balance`, `Cost`, `Salary`, `Total_Due`, `Rate` in financial contexts; `Float` arithmetic with `0.01`.

**Apply.** `assisted`.

**Source.** Ada 95 Rationale §F.1; [POR-10](10-portability.md).

---

## RAT-09 Random number generators: one per task, each explicitly reset

**Rule.** Where several tasks need random sequences, give each its own `Generator` and `Reset` it with a distinct initiator tied to the task's logical function; by default every generator starts in the same state, which "is probably not what is desired". Do not exploit an implementation never yielding 0.0 or 1.0. Shared use of one generator needs synchronisation ([CON-14](09-concurrency.md)).

**Detect.** `auto`. One package-level `Generator` used from several task bodies; `Reset (G)` with no initiator in task-parallel code.

**Apply.** `assisted`.

**Source.** Ada 95 Rationale §A.3; ARM §A.5.2.

---

## RAT-10 Explicit `null;` and qualification where intent could be doubted

**Rule.** Write `when Others_Alternative => null;` explicitly rather than leaving what "could be taken as an unintentional omission". Resolve ambiguous aggregates, literals and overloaded calls by qualification (`Point'(X => A, Y => B)`, `Put (Fruit'(Orange))`) or by naming the package (`Botany.Put (Orange)`); qualify an aggregate that mixes `others` with named associations so its bounds are certain. In case choices prefer subtype names (`when Workday =>`) to literal lists or ranges; consider a constant lookup array when a `case` merely maps values.

**Detect.** `auto`. `when \w+ =>\s*when` (empty alternative is illegal anyway); `when others =>\s*end case` (illegal); `case` statements whose every arm is `X := literal` (lookup-array candidate).

**Apply.** `auto` for qualification; `assisted` for lookup arrays.

**Source.** Ada 83 Rationale §3.5, §3.7, §3.11, §4.5, §7.2, §11.5; [STM-11](06-statements.md).

---

## RAT-11 Read `out` parameters directly; declare indefinite objects from their initial value

**Rule.** Compute into the `out` parameter itself rather than a local result copied at the end: introducing the local "is error prone, because the final assignment may be mistakenly omitted". Declare a `String` or other indefinite object directly from its initial value (`S : String := Get_Message;`) instead of the Ada 83 detour through a constant and a sized variable. Assign `out`/`in out` parameters only once no exception can be raised ([REU-08](11-reusability.md)).

**Detect.** `auto`. `Result : T;` in a procedure with an `out` parameter assigned only as `Param := Result;` at the end; `Len : constant Natural := S'Length; Copy : String (1 .. Len) := S;`.

**Apply.** `auto`.

**Source.** Ada 95 Rationale §6.1, §II.12, §3.4.

---

## RAT-12 Simple strong typing with derived types and explicit conversion chains

**Rule.** For quantities that must not be mixed, derive distinct types from a common parent (`type Dollar is new Currency; type Franc is new Currency;`) rather than declaring everything as the parent; keep the parent and its operations in a package so they are inherited; put typed constants on the specific type (`Cent : constant Dollar := 0.01;`), not named numbers that convert silently; show conceptual steps with explicit conversions (`Franc (Mark_To_Franc * Currency (A))`), which cost nothing at run time and "make the intent more explicit". Where an inherited operation is meaningless (`"*"` on `Length` yielding `Length`), define the right one or hide it, but do not over-engineer: "the fact that any specific mechanism does not achieve one hundred percent safety does not mean that this mechanism should be neglected". A private type is elegantly implemented as a derived type of its representation (`return T (F (Rep (X)))`); inside a redefined operator, convert before comparing or the call is recursive.

**Detect.** `auto`. Several objects of one numeric type named for different units (`_Dollars`, `_Metres`, `_Seconds`); named numbers used with several different types.

**Apply.** `advisory`.

**Source.** Ada 83 Rationale §7.3, §7.6; [TYP-03](05-types.md).

---

## RAT-13 Opaque pointers: a private type completed as an access to a type declared in the body

**Rule.** When clients must be insulated from a representation that may change, implement the private type as `type Hidden; type Opaque is access Hidden;` in the private part with `Hidden` completed in the body; this is "the one case where full type information can indeed be deferred until the package body". Give the full declaration of a limited private type default component values so objects are always initialised ("the only safe way"). Export a read-only view of a variable with a deferred constant of an access-to-constant type.

**Detect.** `assisted`. Private types whose full view is a large record frequently changed; limited private record types with uninitialised components.

**Apply.** `advisory` for the opaque pointer (it introduces allocation); `auto` for default component values.

**Source.** Ada 83 Rationale §9.2, §9.3; Ada 95 Rationale §7.2.

---

## RAT-14 Generics: minimal formal assumptions, box defaults, signature packages

**Rule.** Declare formal types with the minimum needed (`type Index is (<>)`, `type Item is private`, `type T (<>) is private` for indefinite actuals): "assuming an integer type would have been overspecification". Give formal subprograms the box default `is <>` for natural operators ("very good programming practice"), so that "in most applications it should be possible to have only types as mandatory parameters". Compose generics with formal packages (`with package C is new Generic_Complex_Numbers (<>);`) instead of re-importing a type and its operations one by one, "which incurred the slight risk that the user might have redefined one of them with incorrect properties"; use signature packages (formals only, empty body) to parameterise by an abstraction with several implementations. In a generic body, initialise a constant from a parameter rather than declaring an uninitialised variable of the formal type, which fails for unconstrained actuals. Never instantiate two generics mutually in each other's bodies. Annotate Ada 83 generics needing indefinite actuals with `-- !! (<>) in Ada 95`.

**Detect.** `auto`. Generics with `type T is range <>` where the body uses only discrete operations; formal operators without `is <>`; generics re-importing `Element_Type`, `"="`, `Hash` alongside a formal package that already has them ([REU-16](11-reusability.md)).

**Apply.** `assisted`.

**Source.** Ada 83 Rationale §12.2, §12.3, §12.4; Ada 95 Rationale §II.11, §12.1, §12.6.

---

## RAT-15 Real-time annex pragmas stated explicitly and visibly

**Rule.** Request the policies a unit's correctness relies on: `pragma Task_Dispatching_Policy (FIFO_Within_Priorities);` (which requires `Locking_Policy (Ceiling_Locking)`), and `pragma Queuing_Policy (FIFO_Queuing);` when FIFO service matters, so a conflict with a `Priority_Queuing` unit is detected; "portable applications should rely only on the Priority_Queuing and FIFO_Queuing policies". Use `pragma Interrupt_Priority`, not `Priority`, for a priority that may block interrupts, so it is "very visible in the source code". On multiprocessors avoid nested protected calls or fix their ordering, since ceiling locking prevents deadlock only on one processor. Under `Max_Storage_At_Blocking`, keep task-local data and entry parameters small and do not block inside deeply nested calls or large-data blocks. Perform mode changes in stages through protected objects of increasing priority.

**Detect.** `auto`. Tasking code with priorities and no `Task_Dispatching_Policy` pragma; `pragma Priority (System.Interrupt_Priority'First ...` instead of `Interrupt_Priority`; nested protected calls in multiprocessor builds.

**Apply.** `assisted`.

**Source.** Ada 95 Rationale §D.1, §D.2, §D.3, §D.5, §D.7; [CON-08](09-concurrency.md), [STR-03](04-structure.md).

---

## RAT-16 Implementation-defined names in a separate child; `with` list portability

**Rule.** Put implementation-defined entities in their own child (`Ada.Interrupts.Names`, `Ada.Characters.Latin_1` are the models) so a `use` on the parent cannot hide user names and so the `with` clauses alone reveal dependence on system-defined features. Use library-unit renaming to hide a parent/child relationship from clients and to hold project-wide abbreviations in one place. Declare new primitive operations only in a package specification; keep dispatching helpers and the derivation chain in the private part when clients need only the entry points ("minimize spurious visibility").

**Detect.** `auto`. Packages mixing standard and vendor entities; `use GNAT.*`; `renames` of the same package in many units ([STR-16](04-structure.md)).

**Apply.** `assisted`.

**Source.** Ada 95 Rationale §A.1, §10.1, §13.5.

---

## RAT-17 Access parameters for tagged types; return class-wide values, not `out` parameters

**Rule.** For operations on tagged types that need to take `'Access` of the operand or to modify it from a function, access parameters (`procedure P (X : access T)`) "pass the correct accessibility level and avoid the risk of an illegal program or unexpectedly raising Program_Error"; use `in` for read-only operations "just to emphasize that they are read only". (Ada 2012's `in out` function parameters and `aliased` parameters supersede much of this; see [MOD-21](14-modern-ada.md).) Return a class-wide value as a function result, since an `out` parameter of a class-wide type must already be constrained by an initial value. "It is often preferable to manipulate access values to tagged types rather than tagged type values themselves"; export pointer-based containers as `limited private` so users cannot copy a structure "which might subsequently become nonsense". Plan hierarchies so common infrastructure (a queue link) is in the root; "object oriented programming requires thought especially if variant programming is to be avoided".

**Detect.** `assisted`. Procedures with `out T'Class` parameters; tagged types copied by value in containers.

**Apply.** `advisory`.

**Source.** Ada 95 Rationale §6.1, §4.4.

---

## RAT-18 Representation: attach to types, use `Bit_Order`, `pragma Pack` for intent only, `Export` what foreign code touches

**Rule.** Attach representation clauses to types, not objects (per-object clauses "have to be duplicated in many separate declarations"), and only when information essential to the choice is known (a hardware interface). Use `pragma Pack` when only storage minimisation matters: "the pragma expresses the intent only, and the program remains portable"; a representation clause leaves the compiler "no choice". Give `for R'Bit_Order use Low_Order_First;` with every record representation clause so bit numbering is known. For a second representation of the same data, derive a type and convert explicitly ([POR-14](10-portability.md)). Apply `pragma Export` (or the aspect) to any object or subprogram that machine-code inserts or external code touch, so the optimiser cannot remove it. Keep machine code in procedures consisting solely of code statements.

**Detect.** `auto`. `for \w+ use record` without `Bit_Order`; `'Size use` on objects; `pragma Pack` next to a full record representation clause (redundant).

**Apply.** `auto` for adding `Bit_Order` matching the current target's default (document it); `assisted` otherwise.

**Source.** Ada 83 Rationale §15.2 to §15.4, §15.7; Ada 95 Rationale §13.1, §C.1, §B.1.

---

## RAT-19 Distribution and interfacing idioms

**Rule.** For the Distributed Systems Annex, categorise units (`Pure`, `Shared_Passive`, `Remote_Types`, `Remote_Call_Interface`), design RCI units "independent of their eventual location", handle `System.RPC.Communication_Error` where a partition may vanish, and pass class-wide values only of types declared in categorised visible parts. For C: `package C renames Interfaces.C; use type C.char_array;`; declare an Ada procedure when the C return value is of no interest; map `struct *` to a record with `Convention (C)`; reclaim `Interfaces.C.Strings` storage with `Free` on the Ada side; a foreign main must call `adainit`/`adafinal`; a callback passed as an access value needs only `Convention`, not `Export`.

**Detect.** `auto`. `Interfaces.C.Strings.New_String` without `Free`; `pragma Export` on callbacks only passed by `'Access`.

**Apply.** `assisted`.

**Source.** Ada 95 Rationale §B.1, §B.2, §E.2, §E.4; [POR-15](10-portability.md).

---

## RAT-20 Declarations: constants emphasised, locals after nested bodies, `T'Base` for intermediates

**Rule.** "Good programming style in Ada will systematically emphasize constants, even for short texts" (`Old_Left : constant Integer := Left;`); initialise in the declaration rather than in a later run of assignments ([NAM-10](02-naming.md), [TYP-17](05-types.md)). Consider declaring a subprogram's local variables after its nested bodies so they cannot be referenced up-level, which also "makes it easier for a compiler to allocate such variables to hardware registers". In generic numeric code declare intermediates as `Float_Type'Base` so the actual's constraints do not apply to working values, and remember that unconstrained floating types carry no range checks, only overflow checks. A fixed-point expression with two multiplying operators needs an intermediate conversion fixing its accuracy.

**Detect.** `auto`. Variables assigned exactly once ([NAM-10](02-naming.md)); locals declared before nested bodies and referenced from them; `Float_Type` (not `'Base`) locals in generic bodies.

**Apply.** `auto` for `constant`; `assisted` for the rest.

**Source.** Ada 83 Rationale §3.1, §3.2; Ada 95 Rationale §3.3, §3.10.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| RAT-01 | `Pure` > `Preelaborate` > `Elaborate_Body` on every library unit; `Elaborate_All` over `Elaborate` | auto |
| RAT-02 | Membership tests or dispatching, never tag comparison | assisted |
| RAT-03 | `Limited_Controlled` handles for paired operations; no semaphores | assisted |
| RAT-04 | `'Valid` rather than membership for external data; `Normalize_Scalars` | auto |
| RAT-05 | `delay until`, `Ada.Real_Time`, `Milliseconds (N)`, named durations | auto/assisted |
| RAT-06 | `Current_Error`, non-generic numeric I/O, save/restore default files | auto/assisted |
| RAT-07 | String kind by storage behaviour | assisted |
| RAT-08 | Decimal fixed point for money | assisted |
| RAT-09 | One random generator per task, explicitly reset | assisted |
| RAT-10 | Explicit `null;`, qualification, subtype names as case choices | auto/assisted |
| RAT-11 | Read `out` parameters directly; indefinite objects from initial values | auto |
| RAT-12 | Derived types for units; explicit conversion chains | advisory |
| RAT-13 | Opaque pointers; default component values for limited private types | advisory/auto |
| RAT-14 | Minimal generic formals, box defaults, formal and signature packages | assisted |
| RAT-15 | Real-time policies stated explicitly; `Interrupt_Priority`; no nested protected calls on multiprocessors | assisted |
| RAT-16 | Implementation-defined names in a child; `with` list portability; minimal spurious visibility | assisted |
| RAT-17 | Access parameters and class-wide results for tagged types; plan the root | advisory |
| RAT-18 | Representation on types; `Bit_Order`; `Pack` for intent; `Export` what foreign code touches | auto/assisted |
| RAT-19 | Distribution categorisation; C interfacing idioms | assisted |
| RAT-20 | Constants emphasised; locals after nested bodies; `'Base` intermediates | auto/assisted |
