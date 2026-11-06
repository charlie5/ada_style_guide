# 7. Subprograms, Parameters and Overloading

Rules `SUB-*`. Source: Ada 95 Quality and Style (Q&S95) §5.2 Parameter Lists, §5.7.3 to §5.7.5 Overloading; Q&S83 §5.2, §5.7. Structural rules about subprograms (single action, function side effects) are [STR-04](04-structure.md) and [STR-05](04-structure.md).

---

## SUB-01 Descriptive formal parameter names

**Rule.** Name formals so that a call with named association reads as prose: `List_Manager.Insert (Element => New_Employee, Into_List => Probationary_Employees, At_Position => 1);`. Apply the object naming rules ([NAM-07](02-naming.md)); prepositions (`Into`, `From`, `At`, `Onto_Top`) are often the right choice for the second and later parameters.

**Detect.** `assisted`. Formals named `A`, `B`, `X`, `Arg1`, `Param`, `Val`, `Obj`, `Item` (when the subprogram has more than one); formal names identical to their type.

**Apply.** `assisted`. Renaming a formal changes every call that uses named association; search the project first.

**Source.** Q&S95 §5.2.1.

---

## SUB-02 Named association

**Rule.** Use named parameter association in calls to subprograms that are called from fewer than five places in the file or that have more than two formals; in every generic instantiation; whenever an actual is a literal or an expression (the name is the only way to say what the literal means); and whenever a non-default value is given to an optional parameter. Positional association is fine for familiar short subprograms (`Swap (A, B)`, `Sin (X)`) and for aggregates with a conventional order. Consequence: formal names become part of the interface and cannot be changed without editing calls.

**Detect.** `auto`. Calls with 3+ positional actuals; calls whose positional actuals include a literal (`\(\s*[0-9]`, `, True`, `, False`, `, "`), unless the subprogram takes one parameter; `is new \w+ \(` with positional actuals.

**Apply.** `auto` when the subprogram's formal names are known (read the spec). Exact transformation.

**Example.**
```ada
Encode_Telemetry_Packet (Source         => Power_Electronics,
                         Content        => Temperature,
                         Value          => Read_Temperature_Sensor (Power_Electronics),
                         Primary_Module => True);
```

**Source.** Q&S95 §5.2.2.

---

## SUB-03 Default parameters at the end; defaults for added parameters

**Rule.** Give widely used subprograms default values for the parameters that most calls agree on, and put the defaulted parameters last so positional callers can omit them. When extending a subprogram during maintenance, give the new parameters defaults that preserve the old behaviour so existing calls need no change. If the defaults needed to recover the old behaviour are unnatural, write a separate subprogram instead.

**Detect.** `auto`. A defaulted formal followed by a non-defaulted one; many calls passing the same actual for one parameter.

**Apply.** `assisted`. Reordering formals breaks positional callers; adding a default is safe.

**Source.** Q&S95 §5.2.3; ARM Annex A for examples.

---

## SUB-04 Explicit modes, the most restrictive that works

**Rule.** Write the mode of every procedure and entry parameter, including `in`, even though `in` is the default. Use the most restrictive mode that reflects the actual use: not `in out` for a parameter that is only read, and not `in out` where `out` suffices (an `out` parameter may be read after it is assigned). Stated exception: a parameter whose mode is `in` only because of a representation choice (a pointer to the real data) may be declared `in out` to document the operation and allow the representation to change.

**Detect.** `auto`. Procedure or entry parameters with no mode: `\(\s*\w+\s*:\s*(?!in\b|out\b|access\b)\w` and `;\s*\w+\s*:\s*(?!in\b|out\b|access\b)\w` inside a procedure profile. `in out` parameters that are never assigned in the body (GNAT `-gnatwk`/`-gnatwm` warn about this).

**Apply.** `auto` for inserting `in`; `assisted` for narrowing `in out` to `in` or `out`, since both are compiler-checkable after the change. Function parameters: the guide's text predates Ada 2012; leave the mode off for function `in` parameters only if that is the project's existing convention.

**Example.**
```ada
procedure Open_File (File_Name   : in     String;
                     Open_Status :    out Status_Codes);
```

**Source.** Q&S95 §5.2.4; Q&S83 §5.2.4.

---

## SUB-05 Overload only for the same action on different types

**Rule.** Overload a subprogram name only when the overloads perform the same action on arguments of different types (`Sin` on radians, degrees, vectors, matrices), and only for widely used operations. Beware that an overload with a non-distinct profile hides rather than overloads the original. Identical names in different packages are not overloading and are fine.

**Detect.** `auto`. Same subprogram name declared more than twice in one spec with profiles differing only in parameter count; overloads whose bodies do visibly different things.

**Apply.** `assisted` rename of the odd one out.

**Source.** Q&S95 §5.7.3.

---

## SUB-06 Overloaded operators keep their conventional meaning

**Rule.** Overload an operator only with its conventional meaning: `"+"` for adding, joining, increasing; `"-"` for subtracting, separating, decreasing. Apply operators sparingly and uniformly to tagged types, where meaning across a hierarchy is easy to misread. Remember that every overload is a maintenance liability when a profile changes.

**Detect.** `auto` candidates: `function "+"`/`"-"`/`"*"`/`"&"` declared on types that are not numeric, string-like or set-like; `"*"` used for anything but multiplication or scaling. `assisted`.

**Apply.** `advisory`.

**Source.** Q&S95 §5.7.4.

---

## SUB-07 Define equality for private types deliberately

**Rule.** For every private type, decide what `"="` should mean and define it: the predefined equality compares the representation, which for an access-type implementation means pointer identity, and for floating-point components means model-interval equality. Redefine to deep equality or epsilon equality as the abstraction demands. Keep `"="` reflexive, symmetric and transitive. If `"="` returns something other than `Standard.Boolean` (three-valued logic, component-wise vector comparison), redefine `"/="` too, since it is then not the complement.

**Detect.** `auto`. `type \w+ is private` whose full type is an access type or a record containing access or real components, with no `function "="` in the spec.

**Apply.** `assisted`. Adding `"="` changes the result of existing comparisons, membership tests and container operations; audit uses.

**Source.** Q&S95 §5.7.5; §7.2.7; Ada 95 Rationale §6.3.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| SUB-01 | Descriptive formal names that read well in named calls | assisted |
| SUB-02 | Named association for rarely called, many-parameter, literal-actual and generic calls | auto |
| SUB-03 | Defaults last; defaults preserve old behaviour when parameters are added | assisted |
| SUB-04 | Explicit `in`; most restrictive mode | auto/assisted |
| SUB-05 | Overload only for the same action on different types | assisted |
| SUB-06 | Operators keep conventional meaning | advisory |
| SUB-07 | Deliberate `"="` for private types; equivalence-relation properties | assisted |
