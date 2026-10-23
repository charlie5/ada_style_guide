# 5. Types and Data Structures

Rules `TYP-*`. Source: Ada 95 Quality and Style (Q&S95) §3.4 Using Types, §5.3 Types, §5.4 Data Structures, §5.9 Erroneous Execution (the unchecked features and initialisation); Q&S83 §3.4, §5.3, §5.4.

The guides' thesis: strong typing is the main way Ada detects logic errors at compile time, and the extra declarations cost almost nothing at run time. Part A is about declaring types, Part B about composing data, Part C about the unchecked and low-level features that bypass the type system.

---

## Part A: Declaring types

## TYP-01 Constrain scalar types to the application's real range

**Rule.** Limit the range of every scalar type as much as the application allows; get the range from the problem, not from the machine. Do not declare counts and indices as bare `Integer` ("I haven't the foggiest idea how many"). Do not redeclare the subtype names of package `Standard`. Use subtypes to name and document constraints.

**Why.** Eliminating meaningless values lets the compiler and run time catch errors, documents the design decision in the declaration, and forces thought about each use. Predefined types have different ranges on different implementations ([10-portability.md](10-portability.md)). Stated exception: an index with no inherent range (a list of unbounded size).

**Detect.** `auto` candidates: `:\s*Integer\b` and `:\s*Natural\b` in object and component declarations; `range\s+Integer'First`; `type \w+ is new Integer;` with no constraint. `assisted` judgment.

**Apply.** `assisted`. Introduce `type X is range L .. H;` or a subtype; every mixed-type expression then needs attention, so compile after each change.

**Example.**
```ada
type    Day_Of_Leap_Year     is                  range 1 .. 366;
subtype Day_Of_Non_Leap_Year is Day_Of_Leap_Year range 1 .. 365;
subtype Card_Image           is String (1 .. 80);
```

**Source.** Q&S95 §3.4.1; §7.2.1.

---

## TYP-02 Enumeration types instead of numeric codes

**Rule.** Represent a closed set of alternatives as an enumeration type, never as integer constants. If an external device or protocol needs particular codes, keep the enumeration and add an enumeration representation clause (`for Color use (...)`) in the interface package, and only when truly necessary, because reordering literals to match a new platform's codes is a portability trap.

**Why.** Enumerations are robust against additions and deletions, give `'Pos`, `'Val`, `'Succ`, `'Pred`, `'Image`, `'Value` for free, and let `case` statements be checked for completeness.

**Detect.** `auto`. Runs of `\w+\s*:\s*constant\s*(Integer|Natural|:=)\s*:=\s*\d+;` with consecutive values; `case` on an integer with literal alternatives.

**Apply.** `assisted`. Replace the constants with a type; conversions to and from the wire format become `'Pos`/`'Val` or a representation clause.

**Source.** Q&S95 §3.4.2.

---

## TYP-03 Derived types and subtypes in concert

**Rule.** Build new types from existing ones by derivation (`type Dollars is new Number; type Cents is new Number;`) so that unrelated quantities cannot be mixed. Define the base type to include the largest useful set of values including sentinels and boundary values, then constrain with subtypes that exclude them, so the body can use the sentinel without conversions while the interface restricts clients. Use derivation (not tagged extension) when there are no components to add.

**Why.** Distinct derived types enforce strong typing; subtypes of one type stay compatible in expressions. Together they remove flag variables, special cases and conversions. The price is that subtypes and derived types all change when the base changes, which is usually what you want.

**Detect.** `assisted`. Several `type X is range ...` with identical ranges that represent related quantities; `Integer (X)` conversions between things that should be one type; flag variables standing in for an out-of-range sentinel.

**Apply.** `advisory`.

**Example.**
```ada
type    Columns       is range First_Column - 1 .. Listing_Width + 1;   --  allows a sentinel
subtype Listing_Paper is Columns range First_Column .. Listing_Width;   --  what clients see
```

**Source.** Q&S95 §5.3.1; Ada 95 Rationale §4.2.

---

## TYP-04 No anonymous array types

**Rule.** Declare a named array type and declare objects of it; do not write `X : array (...) of T;`. An anonymous array object cannot be passed to a subprogram (no type mark for the formal), and two objects declared with the same anonymous type are of different types. Stated exception: a one-off table that will never be a parameter, such as the periodic table. Access parameters and access discriminants are the deliberate use of anonymous types: they make the designated object behave as a constant for the subprogram or object.

**Detect.** `auto`. `:\s*array\s*\(` in object declarations (not in `type ... is array`).

**Apply.** `auto`. Introduce `type X_Array is array (...) of T;` next to the object and use it. Safe unless the object is used with another anonymous array, which was already a type error.

**Source.** Q&S95 §5.3.2; §6.1.4 for anonymous task types.

---

## TYP-05 Private, limited private, controlled: hide as much as possible

**Rule.** Prefer a private type over a visible record; a limited private type over a private one; and, when assignment and equality must have deep-copy or deep-compare semantics, a private type derived from `Ada.Finalization.Controlled` with `Adjust`, `Finalize` and `"="` overridden, rather than making the type limited. When a client needs one operation that privacy forbids, export that operation explicitly instead of dropping a level of restriction.

**Why.** The more restricted the type, the more the implementation can change without touching clients. Limited private permits only membership, selected discriminants, qualification, conversion, `'Base`, `'Size`, `'Constrained`. Note that `Direct_IO` and `Sequential_IO` cannot be instantiated with a limited type.

**Detect.** `auto` candidates: `type \w+ is\s*record` in a package spec's visible part (not private); `type \w+ is private` implemented with an access type but no redefined `"="`. `assisted`.

**Apply.** `assisted`. Making a visible record private requires accessors for every component clients touch; find them with a project-wide search.

**Source.** Q&S95 §5.3.3, §5.4.5, §5.7.5; §8.3.3 for generics.

---

## TYP-06 Dynamic dispatch before access-to-subprogram

**Rule.** Use access-to-subprogram types for genuine indirection: callbacks, iterators, numerical algorithms, state machine tables. For dynamic selection of behaviour, prefer an abstract tagged type with a dispatching operation and an access-to-class-wide type; it gives more flexibility and type safety.

**Detect.** `assisted`. `type \w+ is access (procedure|function)` used as a record component alongside an enumeration tag; arrays of access-to-subprogram indexed by a kind.

**Apply.** `advisory`.

**Source.** Q&S95 §5.3.4; Ada 95 Rationale §3.7.2.

---

## Part B: Composing data

## TYP-07 Discriminants as constrained as possible

**Rule.** Give a discriminant the most specific subtype you can. `Count : Natural := 0` constraining an array component lets an unconstrained object of the type demand `Natural'Last` elements and fail at run time; `Count : Max_Numbers := 0` with `subtype Max_Numbers is Natural range 0 .. 42` is what was meant. Use a discriminated record, not a constrained array padded with unused slots, when the real length varies.

**Detect.** `auto`. `\(\s*\w+\s*:\s*(Natural|Positive|Integer)\s*(:=\s*\w+)?\s*\)\s*is\s*record` followed by an array component constrained by that discriminant.

**Apply.** `assisted`. Introduce a subtype with the real maximum; recompile.

**Source.** Q&S95 §5.4.1.

---

## TYP-08 Records for related data; no parallel arrays

**Rule.** Group heterogeneous related data into a record, and use arrays of records rather than parallel arrays indexed in step. Use records (with representation clauses where needed) to map device data. Stated exception: hardware that forces the data to be scattered.

**Why.** Related data in one construct is found and updated together, by aggregate where possible; parallel arrays are a Fortran artefact and are easy to update inconsistently.

**Detect.** `auto` candidates: two or more arrays with identical index subtype declared together (`Names : array (Employee_Range) of ...; Ages : array (Employee_Range) of ...`); loops indexing several arrays with the same index.

**Apply.** `advisory`.

**Source.** Q&S95 §5.4.2.

---

## TYP-09 Tagged types and dispatching instead of variant records plus case

**Rule.** For heterogeneous collections, use an access-to-class-wide type and type extension, not a variant record with an enumeration discriminant and `case` statements scattered through the program. Stated exception: when adding new operations is far more common than adding new variants (graphics output is the guide's example), the variant record keeps each operation in one place.

**Why.** With variant records, the expertise about each variant is spread across every `case`; with tagged types it is localised in the type's primitives, and adding a variant touches one place.

**Detect.** `auto` candidates: `case \w+\.Kind is` (or `.Tag`, `.Sort`, `.Variant`) appearing in several subprograms on the same discriminant. `assisted`.

**Apply.** `advisory`. This is a refactoring of the data model.

**Source.** Q&S95 §5.4.3, §5.6.3, §9.2.1.

---

## TYP-10 Factor record structures; name for readable paths

**Rule.** Records need not be flat: factor common parts into their own record types, group related components of a large record into sub-records, and choose component names that read well when nested (`Window.Bottom_Right.Row`). Consider type extension rather than composition when the intermediate records are members of one abstraction family.

**Detect.** `assisted`. Records with more than about 15 components, or with component name prefixes that suggest a grouping (`Pos_X`, `Pos_Y`, `Size_W`, `Size_H`).

**Apply.** `advisory`.

**Source.** Q&S95 §5.4.4.

---

## TYP-11 Dynamic allocation with discipline

**Rule.** Use allocators only when objects must be created and destroyed dynamically or referenced by several names. Then:
- Initialise every access variable and access component explicitly, even though the default is `null`.
- Never drop the only pointer to an undeallocated object; never leave a dangling reference after deallocation.
- Deallocate explicitly with `Ada.Unchecked_Deallocation` (it documents the decision to abandon the object) or keep a free list; do not rely on the environment reclaiming storage.
- Specify the pool size with a `'Storage_Size` clause and provide handlers for `Storage_Error`.
- Implement private types that own dynamic data as controlled types so `Finalize` releases storage and `Adjust` gives copy semantics; redefine `"="` for deep comparison.
- Unless the run time reliably reclaims heap, declare access types, unconstrained composite objects, objects with non-static bounds, large composite objects and task arrays only in the outermost declarative part of a library package, the main subprogram or a permanent task, and avoid unconstrained record objects and non-static task arrays. A large composite may still be an `in` or `in out` parameter.

**Why.** Dangling references and dropped pointers corrupt unrelated code and the allocator itself; leakage and fragmentation lead to `Storage_Error` from which recovery usually means restarting. Controlled types make release automatic instead of trusting clients to call a deallocator.

**Detect.** `auto`. `new\s+\w+` with no `Unchecked_Deallocation` instantiation in the same unit; `Unchecked_Deallocation` called on a pointer that is also stored elsewhere; access components with no initialiser in record declarations; `exception` sections lacking `when Storage_Error` in units that allocate; access types declared inside subprograms.

**Apply.** `assisted`. Adding `:= null` initialisers is safe. Adding deallocation needs ownership analysis.

**Source.** Q&S95 §5.4.5, §5.9.2; §5.9.1, §6.1.5, §6.3.2.

---

## TYP-12 Minimise aliasing

**Rule.** Use `aliased` sparingly: for statically created ragged arrays of `aliased constant` strings referenced through an access-to-constant type, and to hand out an access to the useful part of a container node while hiding the links. Prefer `access constant` when the referenced object must not change. Use access discriminants for self-referential structures, where the permanent binding to the enclosing object is exactly what is wanted.

**Detect.** `auto`. `aliased\s+(?!constant)` on variables; `'Access` on local variables.

**Apply.** `assisted`.

**Source.** Q&S95 §5.4.6, §5.4.7; Ada 95 Rationale §3.7.1, §4.6.3.

---

## TYP-13 Modular types for bit manipulation

**Rule.** Use a modular type (or `Interfaces.Unsigned_N`) rather than a packed Boolean array when the data needs `and`, `or`, `xor`, shifts, or a fixed width smaller than a word. Boolean arrays are fine when the width is not known in advance and performance does not matter.

**Detect.** `auto`. `array (...) of Boolean` with `pragma Pack` used with `and`/`or`; integer types used with `and`/`or` through `Unchecked_Conversion`.

**Apply.** `assisted`.

**Source.** Q&S95 §5.4.8; §10.6.3.

---

## Part C: Unchecked and low-level features

## TYP-14 Unchecked_Conversion, Unchecked_Deallocation, Unchecked_Access isolated in bodies

**Rule.** Instantiate and use `Ada.Unchecked_Conversion` and `Ada.Unchecked_Deallocation` only inside package bodies, never in specs or client code. Check the result of an unchecked conversion of a scalar with `'Valid` before using it; for non-scalars, make sure the bit pattern is a legal value of the target subtype. Use `'Unchecked_Access` only on library-level data, isolated in bodies, and never to return an unsafe value to clients; the one sanctioned use is inserting a local object into a global list from which it is removed before scope exit.

**Why.** All three can produce values that violate type constraints or references to dead objects; the Reference Manual calls the result erroneous execution, whose effects are unpredictable.

**Detect.** `auto`. `Unchecked_Conversion`, `Unchecked_Deallocation`, `Unchecked_Access` in a `.ads` file or outside a `package body`; an unchecked conversion to a scalar type whose result is not followed by `'Valid`.

**Apply.** `assisted`. Move the instantiation into the body and wrap it in a checked function; add the `'Valid` test.

**Example.**
```ada
Possible_Color := Integer_To_Color (Number);
if Possible_Color'Valid then
   ...
else
   raise Invalid_Color_Code;
end if;
```

**Source.** Q&S95 §5.9.1, §5.9.2, §5.9.3; ARM §13.9, §13.9.1, §13.10.

---

## TYP-15 Validate external scalar data with 'Valid

**Rule.** Apply `'Valid` to scalars that arrive from outside the type system: unchecked conversions, input (including `Direct_IO` and `Sequential_IO`, which need not raise `Data_Error` when the check is complex), foreign-language interfaces, objects at a specified `'Address`, and values whose assignment may have been aborted or disrupted.

**Detect.** `auto`. `Sequential_IO`/`Direct_IO` `Read` of a scalar or record-with-scalar with no subsequent `'Valid`; `pragma Import`ed functions returning enumerations.

**Apply.** `assisted`.

**Source.** Q&S95 §5.9.1, §5.9.7; ARM §13.9.2.

---

## TYP-16 Address clauses only for hardware; interrupts via protected procedures

**Rule.** Use `for X'Address use ...` to map an object to a device register or a fixed memory location, never to overlay two Ada objects in Fortran `EQUIVALENCE` style. Make sure the address is valid and meets the type's alignment. Do not put address clauses on program units other than imported ones. Attach interrupt handlers with `Ada.Interrupts` and a protected procedure (`pragma Attach_Handler` / `Interrupt_Handler`, or the aspects in Ada 2012), not with address clauses on task entries, which is obsolescent (ARM Annex J).

**Detect.** `auto`. `for \w+'Address use \w+'Address` (overlay); `for \w+'Address use` on an entry; two objects at the same address constant.

**Apply.** `assisted`. Overlays are usually better written as an `Unchecked_Conversion` in a body or a record with a representation clause.

**Source.** Q&S95 §5.9.4; Ada 95 Rationale §C.3.2.

---

## TYP-17 Initialise before use; beware elaboration

**Rule.** Initialise every object before use, preferably at its declaration; give record components default values in the type. Do not depend on initialisation the language does not define (memory is not guaranteed to be zero). Derive from a controlled type and override `Initialize` when initialisation must be automatic. Be careful with allocators in declarations: `Storage_Error` raised there propagates to the caller, not to the unit's own handler. Do not call a function in a declaration unless it has certainly been elaborated; a call to an unelaborated function raises `Program_Error`, and any exception from a function in a declarative part escapes the unit's handlers. Use `pragma Elaborate_All` / `Elaborate_Body` (or the aspects) to fix elaboration order, not luck.

**Detect.** `auto`. Object declarations of scalar type with no initialiser that are read before any assignment (needs flow analysis; GNAT `-gnatwv` and `-gnatVa` help); `:=\s*\w+\s*\(` calls in package-level declarations of the same package body that declares the function later.

**Apply.** `assisted`. Adding initialisers to scalars is safe; `pragma Elaborate_All` additions can create elaboration cycles, so build after each.

**Source.** Q&S95 §5.9.6; §9.2.3; Ada 95 Rationale §10.3.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| TYP-01 | Constrained scalar types from the application; no bare `Integer` | assisted |
| TYP-02 | Enumerations, not numeric codes | assisted |
| TYP-03 | Derived types for distinct quantities, subtypes for ranges; sentinels in the base | advisory |
| TYP-04 | No anonymous array types | auto |
| TYP-05 | Private > limited private > controlled; export operations rather than loosen | assisted |
| TYP-06 | Dispatching before access-to-subprogram for dynamic selection | advisory |
| TYP-07 | Discriminants as constrained as possible | assisted |
| TYP-08 | Records, not parallel arrays | advisory |
| TYP-09 | Type extension, not variant records plus `case` | advisory |
| TYP-10 | Factored, nested records with readable paths | advisory |
| TYP-11 | Disciplined dynamic allocation; controlled types own dynamic data | assisted |
| TYP-12 | Minimal aliasing; `access constant`; access discriminants for self-reference | assisted |
| TYP-13 | Modular types for bit operations | assisted |
| TYP-14 | Unchecked features isolated in bodies; `'Valid` after conversion | assisted |
| TYP-15 | `'Valid` on external scalar data | assisted |
| TYP-16 | Address clauses only for hardware; protected interrupt handlers | assisted |
| TYP-17 | Initialise before use; control elaboration | assisted |
