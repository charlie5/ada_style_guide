# 13. Performance

Rules `PRF-*`. Source: Ada 95 Quality and Style (Q&S95) chapter 10; Q&S83 chapter 9.

The guide's own warning: "Use the guidelines in this chapter with care; they may be hazardous to your software." Every rule is conditioned on *measured* performance on the actual target: the benefit must outweigh the cost in clarity, maintainability and portability, and what is faster on one compiler may be slower on another. A style reviewer therefore never *applies* these; it reports a performance-motivated construct that lacks a measurement justification, or suggests one of these alternatives only when the user has said performance is the problem. Several of them are the reverse of readability rules elsewhere in this guide ([TYP-08](05-types.md) versus `PRF-04`, [STM-18](06-statements.md) versus `PRF-05`); the readability rule wins unless a measurement says otherwise.

The second, unconditional message of the chapter is that performance comes from design: overall throughput is usually set by the interaction between concurrent activities and peripheral response times, not by the speed of individual statements.

---

## PRF-00 Measure first; document the measurement

**Rule.** Before any performance-motivated deviation from the rest of this guide, measure on the target system (the guide cites the Ada Compiler Evaluation System, ACES, which covers execution speed, code size, compile and link time). Record in a comment at the deviation what was measured and what the alternative cost ([COM-06](03-comments.md), [COM-04](03-comments.md)). Review such comments when the compiler or target changes.

**Detect.** `auto`. Comments containing `performance`, `faster`, `speed`, `optimi[sz]` next to constructs this guide otherwise discourages (parallel arrays, `pragma Suppress`, hand-unrolled loops, `goto`, `Unchecked_Conversion`) with no reference to a measurement.

**Apply.** `assisted`. Ask for the measurement or mark the comment as unverified.

**Source.** Q&S95 §10.1, §10.2.

---

## PRF-01 Late initialisation in blocks

**Rule.** When measured, declare and initialise objects in a block just before their use rather than at the top of the subprogram; it gives the compiler more freedom in register allocation. Some compilers penalise blocks instead; measure.

**Source.** Q&S95 §10.3.1; [STM-17](06-statements.md) gives the readability reason for the same change.

---

## PRF-02 Data structures: constrained arrays, zero-based indexing, fixed-size record components

**Rule.** When measured: use arrays with static bounds (named constants or static expressions) instead of bounds computed at run time; use a lower bound of 0 (or 1, depending on the compiler); replace discriminant-dependent components by fixed-size components, since some compilers allocate the maximum for every object and others put the component on the heap.

**Source.** Q&S95 §10.4.1 to §10.4.3; NASA 1992.

---

## PRF-03 Arrays of records versus parallel arrays

**Rule.** When measured, an array of records, a record of arrays, or parallel arrays may differ significantly in speed. The readability default is the array of records ([TYP-08](05-types.md)).

**Source.** Q&S95 §10.4.4.

---

## PRF-04 Aggregates versus assignment sequences

**Rule.** A static aggregate (size and components known at link time) is generally faster than component assignments; a dynamic aggregate may be slower because it needs a temporary. Extension aggregates (`(Parent_Var with C3 => ..., C4 => ...)`) may cost more than a simple aggregate naming every component. Measure before departing from [STM-18](06-statements.md).

**Source.** Q&S95 §10.4.5, §10.6.1.

---

## PRF-05 Algorithms: `mod`/`rem`, short-circuit forms, `case`, hand checks, array order, overwriting, slices, dispatching

**Rule.** When measured:
- replace `I mod Modulus` in a loop by an incrementing counter reset at the modulus;
- choose between nested `if`s and `and then`/`or else` (correctness, not speed, decides when the second operand depends on the first: [STM-07](06-statements.md));
- prefer `case` to an `if`/`elsif` chain when the compiler produces a jump table (it is also the readability choice, [STM-11](06-statements.md)), and consider a table-driven form ([REU-18](11-reusability.md));
- replace an exception handler for `Constraint_Error` by an explicit range test when the exception mechanism is slow (also the readability choice, [EXC-06](08-exceptions.md));
- process two-dimensional arrays in the storage order the compiler uses, usually row-major, but this is not required by the language;
- assign the common value unconditionally and overwrite it in the `if` for the other case;
- shift packed Boolean arrays with slice assignment rather than element loops, or better use a modular type ([TYP-13](05-types.md));
- dispatch dynamically rather than through an explicit `if`/`elsif` on the tag; the guide expects dynamic dispatch to be faster almost always, so the "static dispatching" alternative exists only to be measured against.

**Source.** Q&S95 §10.5.1 to §10.5.8.

---

## PRF-06 Types: protected objects, modular types, bounded strings, procedural string operations, constrained subtypes

**Rule.** When measured: protected objects rather than a monitor task for mutual exclusion, and a protected procedure for an interrupt handler (both also the design advice of [09-concurrency.md](09-concurrency.md)); modular types rather than packed Boolean arrays for bit sets; `Ada.Strings.Bounded` rather than `Unbounded` where the heap's unpredictable delays matter; the procedural forms of the string operations (`Move`, procedure `Trim`, procedure `Replace_Slice`) rather than the functional ones when copying dominates; and strong typing with constrained subtypes, which lets the compiler drop checks inside tight loops and move them to the input boundary (`Input : constant Name := Get_Response;` with `Name` a constrained `String` subtype). Where both rendezvous and protected objects are too slow, consider `Ada.Synchronous_Task_Control` and `Ada.Asynchronous_Task_Control` from the Real-Time Annex.

**Source.** Q&S95 §10.6.2 to §10.6.7.

---

## PRF-07 Pragmas: `Inline`, `Restrictions`, `Preelaborate`, `Pure`, `Discard_Names`, `Suppress`, `Reviewable`

**Rule.** Use `pragma Inline` (or the aspect) only when measurement shows call overhead is a significant part of a small routine's time; it creates a compilation dependency on the body, may grow code and paging, may defeat other optimisations, and a routine that was small may not stay small. Use `pragma Restrictions` to state restrictions the program obeys, which lets the run time be simpler. Use `Preelaborate` and `Pure` wherever the unit allows them (fewer writes after load; calls whose results are unused may be omitted). Use `Discard_Names` when `'Image` and exception names are not needed and space matters. Use `Suppress` only as [EXC-10](08-exceptions.md) permits. Use `Reviewable` when the generated code must be analysed.

**Detect.** `auto`. `pragma Inline` on subprograms over about 20 lines or with loops; library packages with no state and no elaboration code that lack `Pure`/`Preelaborate` (the reliable test is to add the pragma and compile; the compiler rejects it where it does not apply).

**Apply.** `auto` for adding `Pure`/`Preelaborate` where the compiler accepts it (strictly beneficial, no semantic change); `assisted` for the rest.

**Source.** Q&S95 §10.7.1 to §10.7.7; ARM §10.2.1, §13.12, §C.4, §C.5, Annex H.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| PRF-00 | Measure first; record the measurement at the deviation | assisted |
| PRF-01 | Late initialisation in blocks | advisory |
| PRF-02 | Static array bounds; zero-based indexing; fixed-size components | advisory |
| PRF-03 | Array layout chosen by measurement | advisory |
| PRF-04 | Aggregates versus assignment sequences by measurement | advisory |
| PRF-05 | Algorithm-level alternatives by measurement | advisory |
| PRF-06 | Protected objects, modular types, bounded strings, constrained subtypes | advisory |
| PRF-07 | `Inline` only when measured; `Pure`/`Preelaborate`/`Restrictions` where allowed | auto/assisted |
