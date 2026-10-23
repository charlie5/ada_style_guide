# 11. Reusability and Generics

Rules `REU-*`. Source: Ada 95 Quality and Style (Q&S95) chapter 8; Q&S83 chapter 8 (which lacked controlled types, child units and generic formal packages, so its manual-initialisation and iterator advice is superseded here).

The guide's premise: a reusable part is an extreme case of a well-written part. These rules tighten earlier ones (naming, comments, exceptions) for code that will be used outside the context in which it was written, and they add the discipline for generics. Apply them to library-like code (packages with no application-specific dependencies, generics, container and utility packages). Applying them to application-specific glue code is overkill; say so rather than reporting them there.

---

## REU-01 Application-independent names; leave the simple name for the instantiation

**Rule.** Name reusable parts and their identifiers with the least restrictive names that still say what the part does and how it behaves: `Bounded_Stack` for the generic, so that an instantiation can be called `Stack` or `Tray_Stack`. Include the behavioural characteristic (bounded, unbounded, protected) and not the implementation. Do not use abbreviations in reusable parts at all, even domain-standard ones (`GMT`); the reuser's domain may read them differently. A reusing application may rename the package locally (`package GMT renames Greenwich_Mean_Time;`).

**Detect.** `assisted`. Generics named for one use (`Employee_List_Generic`); abbreviated identifiers in generic specs; generic names that collide with the obvious instantiation name.

**Apply.** `assisted` rename.

**Source.** Q&S95 §8.1.1, §8.1.2.

---

## REU-02 Document generic formal parameters as a contract

**Rule.** Comment each generic formal parameter with what the generic expects of it, beyond what the Ada formal expresses: what a formal `Should_Precede` compares, whether it is a strict order, whether a formal `Swap` must handle aliasing, what the postcondition of the generic is in terms of the formals. (In Ada 2012 and later, express what can be expressed as `Pre`/`Post` on formal subprograms, see [14-modern-ada.md](14-modern-ada.md).)

**Detect.** `auto`. `generic` formal subprogram parameters (`with function`, `with procedure`) and formal types with no adjacent comment.

**Apply.** `assisted`.

**Source.** Q&S95 §8.1.3.

---

## REU-03 Robustness through named numbers and unconstrained arrays

**Rule.** Link dependent sizes to a small number of named numbers through static expressions ([NAM-10](02-naming.md)), so a reuser changes one line. Declare array formals and function results with unconstrained array types, and size local workspace from the actuals (`Workspace : Matrix (Data'Range (1), Data'Range (2))`), so the part adapts to any size.

**Detect.** `auto`. Subprograms in reusable packages whose array parameters have constrained subtypes (`String (1 .. 80)`); locals sized by literals rather than by `'Length`/`'Range` of a parameter.

**Apply.** `assisted`.

**Source.** Q&S95 §8.2.1, §8.2.2.

---

## REU-04 Minimise, enforce and document assumptions

**Rule.** Make as few assumptions as possible. Enforce the ones that remain through subtypes on the interface (`Binary_Value : in Binary_Values` with `subtype Binary_Values is Natural range 0 .. 9_999`), which replaces both the comment and a hand-written check and is often cheaper. Where a subtype cannot express it, add an explicit check that raises a documented exception. Document every assumption, including dependence on a Specialized Needs Annex. Remember that a generic body must cope with any subtype the instantiator chooses.

**Detect.** `auto`. Comments of the form `-- Assumption:`, `-- Assumes`, `-- must be`, `-- caller must` on parameters whose subtype does not enforce the statement; `Natural`/`Integer` parameters immediately range-checked in the body.

**Apply.** `assisted`. Replace the comment with a subtype where the range is static; otherwise keep the check and name the exception in the spec header.

**Source.** Q&S95 §8.2.3.

---

## REU-05 Generic formal subtypes: constraints come from the actual, not the formal

**Rule.** For a generic formal object of mode `in out`, and for the parameters and results of generic formal subprograms, the constraints checked at run time are those of the *actual*, not the subtype written in the formal. So: use first subtypes (base-type names) in those positions to avoid the illusion of a constraint; never assume the freedom of the full base type inside the body; and refer to values through attributes (`'First`, `'Last`, `'Succ`) rather than literals (an index may not start at 1). Adding a formal type parameter does not remove the problem, because the actual object may be of the actual type's base.

**Detect.** `auto`. Generic formal `in out` objects and formal subprogram profiles whose subtype mark is a constrained subtype; literals assigned to generic formal objects (`Object := 1;`); `'First`-less loops over generic formal arrays (`for I in 1 .. `).

**Apply.** `assisted`.

**Source.** Q&S95 §8.2.4; ARM §12.4, §12.6.

---

## REU-06 Do not overload exported names on a formal type in a generic

**Rule.** In a generic, do not declare two subprograms with the same name whose profiles differ only in a formal type versus a concrete type (`Put (Integer)` and `Put (Item)`): an instantiation with `Item => Integer` makes every call ambiguous and the generic unusable for that type. Give them distinct names.

**Detect.** `auto`. Overloaded subprograms in a generic spec where one profile uses a formal private type and another a predefined type in the same position.

**Apply.** `assisted` rename.

**Source.** Q&S95 §8.2.5; ARM §12.3.

---

## REU-07 Document hidden tasks and concurrent access

**Rule.** In a specification, document any task that becomes active merely by withing or using the unit; which generic formal parameters are read or called from a task hidden in the generic (possible unplanned concurrent access to the actuals); and any multithreaded component. Real-time reusers must be able to analyse scheduling.

**Detect.** `auto`. Package bodies declaring tasks whose specs do not mention tasking; generic bodies with tasks that call formal subprograms.

**Apply.** `assisted` header comment.

**Source.** Q&S95 §8.2.6; [STR-14](04-structure.md).

---

## REU-08 Exceptions in reusable parts: propagate, after cleanup, with state intact

**Rule.** Low-level reusable parts propagate exceptions rather than handling them, because they cannot know the right response; handle inside only when the handling is right in every conceivable use. Exceptions raised by a generic formal subprogram are always propagated, after the generic restores its own data structures so later calls work: wrap every call of a formal subprogram that may leave state modified in a `when others => cleanup; raise;` block. A reusable part does all of its function or none: check everything that can fail before changing state, or restore state before propagating; where even that is impossible, document the hazard in the spec header. Assign `out` and `in out` parameters only once no exception can be raised, because copy-back does not happen on exception but by-reference actuals are already modified.

**Detect.** `auto`. Calls to generic formal subprograms with no enclosing handler in bodies that modify state; `out` parameters assigned before validation code that may raise.

**Apply.** `assisted`.

**Example.**
```ada
Catch_Exceptions_Generated_By_Get :
   begin
      Get (Local);
   exception
      when others =>
         Perform_Cleanup_Necessary_For_Process_Numbers;
         raise;
   end Catch_Exceptions_Generated_By_Get;
```

**Source.** Q&S95 §8.2.7, §7.1.8; [EXC-04](08-exceptions.md).

---

## REU-09 Complete core functionality; controlled types for initialisation and finalisation

**Rule.** Give an abstraction its complete core set of operations (so reusers extend rather than patch): initialise, finalise, reset, query operations that let the user avoid exceptions (`Is_Full`), and so on. Any data structure that may hold dynamic data gets initialisation and finalisation; prefer deriving from `Ada.Finalization.Controlled` or `Limited_Controlled` so the compiler inserts the calls, and call the parent's `Initialize`/`Finalize` when overriding. If an implementation links every subprogram whether used or not, a trimmed copy with unused operations commented out is the stated workaround.

**Detect.** `auto`. Private types whose full view contains access components and which are not controlled and have no `Finalize`/`Clear`/`Destroy`/`Free` operation; packages exporting `Push`/`Pop` without `Is_Empty`/`Is_Full`.

**Apply.** `assisted`. Making a type controlled changes it to a tagged type, which affects clients that use it in untagged contexts; check the uses.

**Source.** Q&S95 §8.3.1, §5.4.5.

---

## REU-10 Generics against duplication; parameterise fully; share common instantiations

**Rule.** When two routines differ only in a type or in the subprograms they call, write one generic and instantiate it twice. Then look for further aspects the two instances share by accident and factor them out as formals so a third instance needs no edit. Put widely useful instantiations into the library alongside the generic. Use generics to encapsulate algorithms (a sort over any element and array type, taking `Should_Precede` and `Swap` as formals with `is <>` defaults).

**Detect.** `auto`. Near-duplicate subprogram bodies differing in type names (token-level diff after identifier normalisation); several instantiations of one generic with the same actuals in different units.

**Apply.** `advisory` for extracting a generic; `auto` for sharing a duplicated instantiation (declare it once in a library package and `with` it).

**Source.** Q&S95 §8.3.2, §8.3.4; §9.3.5.

---

## REU-11 Generic formal types: as limited as the body allows; exported types as free as the abstraction allows

**Rule.** Choose the formal type by what the body needs: `limited private` when the body never assigns or compares objects of the type (then any type can be the actual); `private` when it needs assignment and `"="`; a formal derived from `Ada.Finalization.Controlled` when it needs assignment with user-defined semantics, adding a formal `"="` if equality must also be user-defined. For a mixin generic that extends a formal tagged type, make the formal `abstract tagged limited private` and export an abstract extension. Note the cases where a limited formal is required (access discriminants, extension with a limited component). For *exported* types the restriction falls on the user, so export the least restrictive type that preserves the abstraction's integrity: a plain private type when copying and comparing are meaningful and there are no access components; a controlled type with overridden `"="` when there are; a non-private type when nothing is lost by revealing it. `Sequential_IO`/`Direct_IO` need non-limited actuals, which affects I/O for limited exports.

**Detect.** `auto`. Generic formal `private` types in generics whose bodies never use `:=` or `=` on the type (could be `limited private`); formal `limited private` types in bodies that do (compile error anyway); exported `limited private` types with no access components and `Copy`/`Assign` operations (could be `private`).

**Apply.** `auto` for widening a formal from `private` to `limited private` when the body compiles; `assisted` otherwise.

**Source.** Q&S95 §8.3.3, §5.3.3, §8.3.8.

---

## REU-12 Abstract data types over abstract data objects; generic ADTs over both

**Rule.** Prefer a package that exports a type (abstract data type, ADT) to one that encapsulates a single object (abstract data object, ADO, or abstract state machine): users can then declare any number of objects, put them in arrays and records, pass them as parameters, and compare or assign them. A parameterless generic ADO is not a substitute: each instance is a different type with none of those capabilities. Prefer a generic ADT (element type and sizes as formals) to a non-generic one; the body is almost identical, with one extra parameter per operation.

**Detect.** `auto`. Packages with package-level state and operations lacking an object parameter (`procedure Push (New_Element : in Element)` with `Stack : Stack_Information;` in the body); `generic package X` with no formals.

**Apply.** `advisory`. Converting an ADO to an ADT changes every client call.

**Source.** Q&S95 §8.3.5.

---

## REU-13 Iterators for complex structures; both forms; safe under modification

**Rule.** A reusable data structure provides iterators. Provide both a passive iterator (a generic `Iterate` taking a `Process` formal, or in Ada 2005 an `Iterate` taking an access-to-procedure or in Ada 2012 a `for ... of` cursor interface) and an active one (an `Iterator` type with `Start`/`More`/`Next`, or the access-discriminant form `type Iterator (S : access Set)`). Decide and document what happens when the structure is modified during iteration (closure algorithms need to see added elements; priority lists may need to exclude lower-priority insertions) and protect against deletion during iteration, for example with an "iteration in progress" flag checked by every modifying operation, which is possible because the type is limited private. Ada 95 note: an access-to-subprogram actual must be a named access type and cannot designate a nested subprogram; Ada 2005 anonymous access-to-subprogram parameters remove that restriction.

**Detect.** `auto`. Packages exporting a container-like limited private type (`Insert`, `Delete`, `Find`) with no `Iterate`/`First`/`Next`/cursor operations; iterators that dereference without checking modification state.

**Apply.** `advisory` for adding iterators; `assisted` for the documentation.

**Source.** Q&S95 §8.3.6; Ada 95 Rationale §3.7.1, §3.7.2, §4.4.4.

---

## REU-14 Localise currency and locale in decimal output

**Rule.** Put the currency symbol, digit separator, radix mark and fill character for `Ada.Text_IO.Editing` picture output in one instantiation of `Decimal_Output` per currency, and use `#` in pictures so field widths do not change with the currency symbol's length.

**Detect.** `auto`. `Default_Currency`, `Default_Separator` or `Default_Radix_Mark` literals outside one locale package; literal `"$"`/`","` in pictures.

**Apply.** `assisted`.

**Source.** Q&S95 §8.3.7.

---

## REU-15 Mixins through abstract tagged formals

**Rule.** To define a reusable unit of functionality that can be mixed into several abstractions, write a generic that takes `type S is abstract tagged private;` and exports `type T is abstract new S with private;` plus operations; serialise the instantiations so each extends the previous instance's exported type, and derive the final concrete type from the last. One extension per generic package.

**Detect.** `assisted`. Hand-copied "feature" components and operations repeated across several tagged types.

**Apply.** `advisory`.

**Source.** Q&S95 §8.3.8, §9.5.1; Ada 95 Rationale §4.6.2.

---

## REU-16 Independence: subsystems as child hierarchies; generic formals instead of `with`

**Rule.** Structure a reusable subsystem as a parent package with context-independent functionality and child packages for operations that belong to particular contexts, so unused children can be dropped. Minimise `with` clauses on reusable parts, especially their specs; where a part needs only a few operations of another unit, take them as generic formal subprograms (or a generic formal package for a whole related set) instead of withing the unit, which both removes the dependency and narrows visibility to exactly those operations. Prefer generic formal packages for composing one generic abstraction from another.

**Detect.** `auto`. Generic units with `with` clauses on application packages; specs of library-like packages with many `with` clauses.

**Apply.** `assisted`. Moving a `with` to a generic formal changes every instantiation.

**Source.** Q&S95 §8.4.1, §8.4.2; Ada 95 Rationale §12.6.

---

## REU-17 `pragma Elaborate_Body` in generic library unit specs; avoid `Elaborate_All` in reusable parts

**Rule.** Put `pragma Elaborate_Body` (or the aspect) in the specification of every generic library unit, so the body is elaborated right after the spec regardless of compilation order and an instantiation cannot hit `Program_Error` from an unelaborated body. Use it also where a recursive dependency between two packages' bodies and specs exists. Avoid `Elaborate_All` in reusable parts because it constrains the configurations in which they can be combined, while noting it gives link-time rather than run-time detection. `Pure` or `Preelaborate` units need neither. A registry package that other units call during elaboration must itself be elaborated early and must depend only on the root types, holding only class-wide pointers.

**Detect.** `auto`. `generic ... package` library specs lacking `Elaborate_Body`/`Pure`/`Preelaborate`; `pragma Elaborate_All` in generic or library-like units.

**Apply.** `auto` for adding `pragma Elaborate_Body` to generic specs (safe unless the body must be deferred for a cycle; compile and bind to confirm).

**Source.** Q&S95 §8.4.3; ARM §10.2.1; Ada 95 Rationale §10.3.

---

## REU-18 Part families, dead-code structure, table-driven parts, predefined string packages

**Rule.** Provide families of parts with similar specifications (bounded and unbounded stacks) so applications substitute members rather than editing code; tagged-type hierarchies give the same effect with one interface and several implementations. Where variants cannot be separated into units, select among them with a static expression (generic formal object in a `case`) so compilers that remove dead code leave nothing behind, and know whether yours does. Write table-driven parts where behaviour can be captured in data; comment the table structure thoroughly. Use `Ada.Strings.Fixed`, `Bounded`, `Unbounded`, `Maps` and `Ada.Characters.Handling` instead of hand-written `Trim`, `To_Upper`, searching and padding.

**Detect.** `auto`. Hand-written functions named `Trim`, `Upper`, `To_Upper`, `Lower`, `Strip`, `Pad`, `Index_Of`, `Starts_With`, `Replace` over `String` in application code; `Character'Val (Character'Pos (C) + Offset)` case conversion.

**Apply.** `auto` for replacing hand-written string utilities whose semantics match the predefined ones (check trimming sides and whitespace definitions); `advisory` for the rest.

**Source.** Q&S95 §8.4.4 to §8.4.7; §9.2.4.

---

## REU-19 Tagged hierarchies decouple general algorithms from specific types

**Rule.** Put common properties and operations in a root tagged type (abstract where the operation must be overridden), derive the specific types in child packages mirroring the hierarchy, and write general algorithms against the class-wide type (`Compute_Wage (Who (E).all)` over an array of `Employee_Ptr`). Such algorithms need no change when new types are added, and clients `with` only the specific packages they need.

**Detect.** `assisted`. General algorithms that `case` on a kind field ([TYP-09](05-types.md)); hierarchies declared in one package.

**Apply.** `advisory`.

**Source.** Q&S95 §8.4.8; §9.2, §9.3.1, §9.3.5, §9.4.1.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| REU-01 | Application-independent, unabbreviated names; simple name left for the instantiation | assisted |
| REU-02 | Generic formals documented as a contract | assisted |
| REU-03 | Named numbers and unconstrained arrays for adaptability | assisted |
| REU-04 | Assumptions minimised, enforced by subtypes, otherwise checked and documented | assisted |
| REU-05 | Generic formal constraints come from the actual; use base subtypes and attributes | assisted |
| REU-06 | No overloads that collapse on instantiation | assisted |
| REU-07 | Document hidden tasks and concurrent access to formals | assisted |
| REU-08 | Propagate exceptions after cleanup; all-or-nothing effect; late `out` assignment | assisted |
| REU-09 | Complete core operations; controlled types for init/finalise | assisted |
| REU-10 | Generics against duplication; shared instantiations | advisory/auto |
| REU-11 | Formal types as limited as possible; exported types as free as possible | auto/assisted |
| REU-12 | ADTs over ADOs; generic ADTs over both | advisory |
| REU-13 | Iterators, active and passive, safe and documented under modification | advisory |
| REU-14 | Locale of decimal output localised | assisted |
| REU-15 | Mixins via abstract tagged generic formals | advisory |
| REU-16 | Child hierarchies for subsystems; generic formals instead of `with` | assisted |
| REU-17 | `Elaborate_Body` on generic specs; no `Elaborate_All` in reusable parts | auto |
| REU-18 | Part families; dead-code structure; tables; predefined string packages | auto/advisory |
| REU-19 | Tagged hierarchies decouple algorithms from types | advisory |
