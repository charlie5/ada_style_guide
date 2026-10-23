# 10. Portability

Rules `POR-*`. Source: Ada 95 Quality and Style (Q&S95) chapter 7; Q&S83 chapter 7; Ada 2022 Annotated Reference Manual (AARM22) Annex J for the current list of obsolescent features.

The guide's four principles: know which constructs vary between implementations; rely on what all relevant implementations share; localise and encapsulate what cannot be avoided; and highlight it in comments. Most rules here are about *where* non-portable code lives and *whether it is marked*, which a reviewer can check mechanically, rather than about forbidding it.

---

## POR-01 No obsolescent features (Annex J)

**Rule.** In code meant to last, do not use features the Reference Manual lists as obsolescent in Annex J, except where Ada 83 compatibility forces it, and then document the use. The Ada 95 list: the short library-unit renamings (`Text_IO` for `Ada.Text_IO`, `Unchecked_Conversion` for `Ada.Unchecked_Conversion`, and so on); the character replacements `!` for `|`, `:` for `#`, `%` for `"`; reduced-accuracy floating-point subtypes (`digits` on a subtype); `'Constrained` on private types; package `ASCII` (use `Ada.Characters.Latin_1`); `Numeric_Error` (use `Constraint_Error`); `at` mod clauses and `at` component clauses; interrupt entries (use protected procedures); the `Storage_Size` attribute as a representation clause. Later revisions added more; see AARM22 Annex J (J.1 to J.15) for the current list, which also covers `pragma Interface`, `pragma Inline`/`Pure`/... where an aspect now exists (the pragmas remain legal), `Ada.Characters.Handling.Is_Character`/`Is_String`, the `'Class` attribute on untagged incomplete types, and the default I/O file handling in `Ada.Text_IO` for `Standard_Output` manipulation.

**Detect.** `auto`. `with\s+(Text_IO|Sequential_IO|Direct_IO|IO_Exceptions|Calendar|Unchecked_Conversion|Unchecked_Deallocation|Machine_Code);` (the Ada 83 names without `Ada.`); `ASCII\.`; `Numeric_Error`; `\bat\s+mod\b` and `\bat\s+\d+\s+range`; `for \w+'Storage_Size use`; `for \w+ use at`; `!` / `%` used as delimiters.

**Apply.** `auto`. Each replacement is exact: add the `Ada.` prefix, `ASCII.NUL` becomes `Ada.Characters.Latin_1.NUL`, `Numeric_Error` becomes `Constraint_Error`, interrupt entries become protected procedures (`assisted`).

**Source.** Q&S95 §7.1.1; ARM Annex J (AARM22 `AA-J-*.html`).

---

## POR-02 Informed assumptions about target limits

**Rule.** State explicitly, in one place, the minimum platform characteristics the program assumes: bits in `Integer`, decimal digits of `Float`, bits in fixed-point types, source line length, bits for root integer expressions, range and `'Small` of `Duration`, decimal scale. Do not assume more than the Reference Manual guarantees: `Duration` covers at least one day with `'Small` at most 20 ms; `Integer` may be 16 bits unless the project restricts targets to 32-bit systems. Do not assume the number or values of `Character` beyond Latin-1.

**Detect.** `assisted`. Uses of `Integer'Last` as "a big number"; `Duration` arithmetic exceeding one day; literals above 32_767 in `Integer` contexts without a declared type.

**Apply.** `assisted`. Declare the quantities with explicit ranges ([TYP-01](05-types.md)) and let the compiler choose the representation.

**Source.** Q&S95 §7.1.2; ARM §9.6.

---

## POR-03 Highlight every non-portable feature in a comment

**Rule.** Each package, subprogram or task that uses a non-portable feature carries a warning comment at its head, and each use describes what is assumed of the feature (which the next porter will not be able to look up in the original vendor's documentation).

**Detect.** `auto`. Units containing `System.`, `pragma Import`, `Interfaces.`, address clauses, `Unchecked_*`, `Machine_Code`, vendor packages (`GNAT.`, `System.OS_Lib`, ...), record representation clauses, with no comment containing `portab`, `implementation`, `WARNING`, `target` or `platform` in the unit.

**Apply.** `auto` for inserting the comment; the content is `assisted`.

**Example.**
```ada
package Memory_Mapped_IO is
   --  WARNING - This package is implementation specific. It uses absolute
   --  memory addresses to interface with the I/O system and assumes a
   --  132-column printer. Change memory mapping and printer details when porting.
```

**Source.** Q&S95 §7.1.3; §3.3.4.

---

## POR-04 Parameterless main; encapsulated `Ada.Command_Line`

**Rule.** The main subprogram is a parameterless procedure. Obtain environment values through `Ada.Command_Line` only, wrapped in one package that fixes argument positions, meanings and defaults, because what `Argument_Count` and `Argument` return (and whether the environment passes arguments at all) is implementation-dependent. Embedded systems should read configuration from a constants package, switches or a downloaded file instead.

**Detect.** `auto`. `function` as main unit; main `procedure` with parameters; `Ada.Command_Line.Argument (` outside a single environment package; raw argument-index literals scattered through the code.

**Apply.** `assisted`.

**Source.** Q&S95 §7.1.4, §6.3.6.

---

## POR-05 Encapsulate implementation and hardware dependencies

**Rule.** Isolate hardware and implementation dependencies in packages whose specifications will not change when porting, and keep one body per target environment. State the objective (speed, size) when the dependency exists for efficiency, so the porter can decide how to meet it elsewhere. Put interrupt-receiving code in such packages. Annex M of the Reference Manual lists the implementation-defined characteristics.

**Detect.** `auto`. `System`, `Interfaces`, `pragma Import/Export/Convention`, `Machine_Code`, address clauses or vendor packages appearing in more than one or two units of a project. (Same signal as [STR-06](04-structure.md), reported once.)

**Apply.** `advisory`.

**Source.** Q&S95 §7.1.5, §7.6.9.

---

## POR-06 Standard annexes before vendor features

**Rule.** Avoid vendor-supplied packages (`GNAT.*`, `System.*` beyond what the standard defines) and vendor extensions to predefined packages. Where a specialised need exists, use the Specialized Needs Annexes (Systems Programming, Real-Time, Distributed, Information Systems, Numerics, Safety and Security) and document the dependency, because not every compiler implements every annex. Avoid implementation-defined pragmas and attributes; prefer the annexes' standard ones. A vendor package supplied as compilable, licensable source that is itself portable is not a portability problem. Stated exception: user-interface and OS-integration code and performance-critical embedded code may need vendor features; isolate them.

**Detect.** `auto`. `with GNAT\.`, `with System\.(OS_Lib|Soft_Links|...)`, `pragma (Unreferenced|Warnings|Annotate|Inline_Always|Linker_Options|...)` (GNAT-specific), `'(Img|Object_Size|Unrestricted_Access|...)` (GNAT-specific attributes), `Ada.Text_IO` vendor-added subprograms. Note: `'Img` was standardised as `'Image` on objects in Ada 2012; `Object_Size` in Ada 2022; prefer the standard spelling.

**Apply.** `assisted`. Many GNAT pragmas have a standard aspect equivalent in Ada 2012 and later ([14-modern-ada.md](14-modern-ada.md)); replace where one exists, isolate the rest.

**Source.** Q&S95 §7.1.6, §7.1.7, §7.6.5, §7.6.9.

---

## POR-07 No dependence on the parameter passing mechanism or on evaluation order

**Rule.** Untagged records and arrays may be passed by copy or by reference. Do not write code whose result depends on which (reading a global that is also an `in out` actual from inside the subprogram, for example). When a subprogram has two parameters of the same type and at least one is writable, either make it correct when both denote the same object or document that they must not. Do not depend on the order of evaluation of actual parameters, operands, or aggregate components, which the Reference Manual leaves arbitrary (`Unique_ID & Unique_ID` may give 1 2 or 2 1). Foreign-language calls that depend on the mechanism go behind an interface package.

**Detect.** `assisted`. Subprograms that read a package-level variable which callers also pass as an `in out` actual; two calls to the same impure function in one expression or call ([STR-05](04-structure.md)).

**Apply.** `assisted`. Hoist impure calls into separate statements with named temporaries.

**Source.** Q&S95 §7.1.8, §7.1.9; ARM §6.2, §6.4.

---

## POR-08 Numeric types: declare ranges and digits, not `Integer` and `Float`

**Rule.** Avoid the predefined numeric types of `Standard`; declare `type X is range L .. H;` and `type Y is digits D;` and let the implementation pick the representation (`type Second_Of_Day is range 0 .. 86_400;` works on a 16-bit machine; `new Integer range 1 .. 86_400` does not). Predefined `Integer`, `Natural` and `Positive` are appropriate for `String` indexing and slicing, for pure numbers such as exponents, and for iteration counts. Need more precision than the assumed minimum? Wrap it in a package with a private type. Prefer the Numerics Annex strict mode when accuracy and performance matter; analyse the precision actually needed and declare the smallest `digits` that gives it; do not push the machine's accuracy limits; comment the numerical analysis. Anticipate the range of sub-expressions: `(3 * X) / 4` may overflow in 16-bit registers even when the result fits.

**Detect.** `auto`. `type \w+ is new (Integer|Float|Long_Float)`; `subtype \w+ is Integer range`; `: Float` / `: Long_Float` in declarations of physical quantities; `digits 15` and higher without a comment.

**Apply.** `assisted`. Changing a derived-from-`Integer` type to a range type is usually transparent; recompile.

**Source.** Q&S95 §7.2.1 to §7.2.6; Ada 95 Rationale §3.3.

---

## POR-09 Real comparisons through model attributes

**Rule.** Compare real values with `<=` and `>=` against a tolerance built from the type's attributes, never `=`, `/=`, `<`, `>` directly: `abs (X - Y) <= abs X * Float_Type'Model_Epsilon` for relative, `abs (X - Y) <= Float_Type'Model_Small` for absolute; use `'Base` attributes for values computed in registers. Zero is not a special case; evaluate each small-value comparison on its own.

**Detect.** As [STM-08](06-statements.md); additionally literal epsilons such as `1.0E-6` in comparisons.

**Apply.** `assisted`.

**Source.** Q&S95 §7.2.7; Ada 95 Rationale §G.4.1, §G.4.2.

---

## POR-10 Decimal types per scale and unit (information systems)

**Rule.** Declare a decimal type per scale and per unit of measure (each currency its own type with explicit conversions), encapsulated per measure category; keep the number of types for unitless data small. Separate range (a subtype constraint matching current needs) from precision (`digits`, sized for growth). Decide explicitly between truncation (Ada's default) and rounding (`'Round`). Avoid decimal arithmetic on compilers without full Information Systems Annex support, which guarantees 18 digits and `Text_IO.Editing`.

**Detect.** `assisted`. One `delta 0.01` type used for several currencies; `Float` used for money.

**Apply.** `advisory`.

**Source.** Q&S95 §7.2.8; ARM §3.5.9, Annex F.

---

## POR-11 Storage control

**Rule.** Do not use `'Storage_Size` clauses to size tasks or pools for portability; the meaning of the value is ambiguous (porters should treat every occurrence as a redesign point). Do not compare access-to-subprogram values; the implementation may consider two values designating the same subprogram unequal. If you must, compare against a constant taken once. Consider an explicit storage pool (`Storage_Pool` clause) for control over allocation policy, mark/release, or bounded memory; choose the simplest storage strategy that meets the requirements.

**Detect.** `auto`. `'Storage_Size use`; `=` or `/=` with operands of an access-to-subprogram type.

**Apply.** `assisted`.

**Source.** Q&S95 §7.3.1 to §7.3.3; ARM §3.10.2, §4.5.2, §13.11.2.

---

## POR-12 Tasking assumptions that do not port

**Rule.** Do not depend on: the activation order of tasks declared in one declarative list; any particular delay being achievable; the execution pattern of tasks for timing; any correlation between `System.Tick` and `Duration`; the order in which guards are evaluated or how an open alternative is chosen; tasks running uninterrupted until a synchronisation point. Use `pragma Priority` only for coarse levels of importance. Avoid `abort`. Do not use unprotected shared variables; `Atomic`/`Volatile` only when forced by run-time deficiencies. Without the Real-Time Annex these are all implementation choices; with it, much becomes defined and portable among annex-conforming implementations.

**Detect.** As in [09-concurrency.md](09-concurrency.md) (`CON-08`, `CON-09`, `CON-14`, `CON-18`); plus comments asserting activation order.

**Apply.** `advisory`.

**Source.** Q&S95 §7.4.1 to §7.4.7.

---

## POR-13 Exceptions across implementations

**Rule.** Do not depend on where a predefined exception is raised or on which condition caused it; a handler for `Constraint_Error` must be prepared for any of its causes. Do not rely on `Ada.Exceptions` beyond the language-defined minimum (`Exception_Information` content is implementation-defined). Never raise an implementation-specific exception; convert any you catch into user-defined exceptions inside the interface package that deals with the vendor feature, so porting does not mean finding every handler.

**Detect.** `auto`. `when (GNAT|System)\.\w+\.\w+ =>` outside interface packages; `raise (GNAT|System)\.`; parsing of `Exception_Information` output.

**Apply.** `assisted`.

**Source.** Q&S95 §7.5.1, §7.5.2; ARM §11.

---

## POR-14 Representation clauses only for interfaces and designed layouts

**Rule.** Prefer algorithms that do not depend on data representation. Use representation clauses when accessing or defining interface data (devices, protocols, foreign languages) or when a design needs a specific layout, not to document assumptions. Do not assume two programs sharing a source file share a representation: without representation clauses the layout may change with compilation order, optimisation level, compiler version or available resources, so independently linked programs share only explicitly represented data. Use `System` constants only to parameterise other machine-dependent constructs (`System.Storage_Unit` to compute bit positions, for instance). Avoid machine-code inserts; if unavoidable, comment what they do and why higher-level attempts failed.

**Detect.** `auto`. `for \w+ use record` and `'Size use` on types not used in `Import`/`Export`, I/O, or `Unchecked_Conversion`; `System.Word_Size`, `System.Max_Int` in application logic; `with System.Machine_Code`.

**Apply.** `assisted`. Removing an unnecessary representation clause is safe when nothing external depends on the layout.

**Source.** Q&S95 §7.6.1 to §7.6.3; ARM §13.5.1.

---

## POR-15 Foreign-language interfaces via `Interfaces` and `Import`, in bodies

**Rule.** Use package `Interfaces` and its children (`Interfaces.C`, `Interfaces.C.Strings`, `Interfaces.Fortran`, `Interfaces.COBOL`) for foreign types. Use `pragma Import` (or the `Import`, `Convention`, `External_Name` aspects) rather than access-to-subprogram types for static calls; access-to-subprogram remains right for callbacks. Isolate every `Import`, `Export` and `Convention` in interface package bodies, exporting Ada-typed operations and Ada exceptions; convert argument types and check preconditions on the Ada side.

**Detect.** `auto`. `pragma Import` or `with Import` in a `.ads`; `Interfaces.C.` types in a spec that is not an interface package; `Import` of a C function called directly from application code.

**Apply.** `assisted`. Wrap each import in an Ada subprogram in the body.

**Example.**
```ada
function Cube_Root (X : Float) return Float is
   function C_Cbrt (X : Interfaces.C.double) return Interfaces.C.double
     with Import, Convention => C, External_Name => "cbrt";
begin
   if X < 0.0 then
      raise Argument_Error;
   end if;
   return Float (C_Cbrt (Interfaces.C.double (X)));
end Cube_Root;
```

**Source.** Q&S95 §7.6.4; Ada 95 Rationale Annex B.

---

## POR-16 Avoid dependence on the unchecked features

**Rule.** Stricter than [TYP-14](05-types.md) when portability is the goal: avoid `Ada.Unchecked_Deallocation` (storage reclamation varies; deallocating early makes later access erroneous), `'Unchecked_Access` (dangling references), and `Ada.Unchecked_Conversion` (an implementation may limit the matched types and the result for mismatched sizes). Legitimate uses: local control of storage in highly iterative or recursive algorithms; low-level and foreign-language code in isolated packages; retrieving the internal code of an enumeration with a representation clause, for which unchecked conversion is the only Ada 95 way (Ada 2022 adds `'Enum_Rep`).

**Detect.** As [TYP-14](05-types.md).

**Apply.** `assisted`.

**Source.** Q&S95 §7.6.6 to §7.6.8.

---

## POR-17 Input/output

**Rule.** Keep `Name` and `Form` strings for `Open`/`Create` as symbolic constants in the implementation-dependency package (formats differ; an empty `Form` does not guarantee portable defaults). Close every file explicitly; the effect of leaving files open at program end is undefined. Never write access values to a file. Use `Sequential_IO`/`Direct_IO` for homogeneous files, which the compiler then enforces, and reserve `Stream_IO` for heterogeneous files. Report run-time errors through `Current_Error`/`Standard_Error`, not standard output. With several I/O tasks, protect updates to `Current_Input`/`Current_Output`/`Current_Error`, which are shared state in `Text_IO`.

**Detect.** `auto`. `Open (.*Form => "` with a literal; `Create (`/`Open (` with no matching `Close` in the unit; `Write (` on an access type or a record containing one; `Put_Line` of error messages (`"Error`, `"Cannot`, `"Can't`) to standard output.

**Apply.** `auto` for `Put_Line (Standard_Error, ...)` on error messages and for adding `Close` in a `Finalize` or an exception-safe block; `assisted` otherwise.

**Source.** Q&S95 §7.7.1 to §7.7.5; ARM §A.7.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| POR-01 | No Annex J obsolescent features | auto |
| POR-02 | Explicit minimum platform assumptions | assisted |
| POR-03 | Warning comment on every non-portable unit and use | auto |
| POR-04 | Parameterless main; `Ada.Command_Line` encapsulated | assisted |
| POR-05 | Dependencies in packages with stable specs, one body per target | advisory |
| POR-06 | Standard annexes over vendor features | assisted |
| POR-07 | No dependence on passing mechanism or evaluation order | assisted |
| POR-08 | `range`/`digits` declarations, not `Integer`/`Float` | assisted |
| POR-09 | Real comparisons via model attributes | assisted |
| POR-10 | Decimal types per scale and unit | advisory |
| POR-11 | No `'Storage_Size`; no access-to-subprogram comparison; consider pools | assisted |
| POR-12 | No tasking assumptions outside the Real-Time Annex | advisory |
| POR-13 | No dependence on where predefined exceptions arise; convert vendor exceptions | assisted |
| POR-14 | Representation clauses only for interfaces and designed layouts | assisted |
| POR-15 | `Interfaces` and `Import` in interface bodies | assisted |
| POR-16 | Avoid the unchecked features | assisted |
| POR-17 | Portable I/O: symbolic Form, explicit Close, no access I/O, Current_Error | auto/assisted |
