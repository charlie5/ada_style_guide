# 2. Spelling and Naming

Rules `NAM-*`. Source: Ada 95 Quality and Style (Q&S95) §3.1 Spelling, §3.2 Naming Conventions; Q&S83 §3.1, §3.2. The capitalisation and digit-grouping guidelines of §3.1.2 and §3.1.3 are presentation, left to the project's formatter and compiler style checks; NAM-03 is retired and its number is not reused.

Naming carries more of a program's readability than comments do. Renames are safe to apply mechanically only when every use site is updated and the name is not part of a published interface; see [00-procedure.md](00-procedure.md) for the rename protocol.

---

## NAM-01 Underscores separate words

**Rule.** Use underscores to separate the words of a compound identifier: `Miles_Per_Hour`, `Entry_Value`. Never run words together (`MilesPerHour`, `milesperhour`).

**Why.** Easier to read, and a formatter can then re-case words reliably.

**Detect.** `auto`. Identifiers matching `[a-z][A-Z]` (camel case) or long all-lowercase identifiers with no underscore that are clearly multi-word.

**Apply.** `assisted` rename.

**Source.** Q&S95 §3.1.1.

---

## NAM-02 Radix and exact fractions in literals

**Rule.** Write each literal in the radix appropriate to the problem: hexadecimal (or octal, binary) for addresses, bit masks and register values, decimal otherwise. Express an exact fraction as a static expression (`1.0 / 3.0`), not as a decimal approximation.

**Why.** A literal in the problem's own radix reads as the quantity it denotes. A static expression is evaluated exactly at compile time, whereas a truncated decimal silently loses accuracy; Q&S95 also notes that a radix in which the fraction terminates (`3#0.1#`) may convert more accurately to the machine base.

**Detect.** `assisted`. Decimal literals used as masks or addresses: `\b(and|or|xor)\s+[0-9]{2,}\b`, `'Address use [0-9]`, `Address => [0-9]`. Real named numbers with a repeating digit run: `0\.(3{4,}|6{4,}|1{4,}|9{4,})`.

**Apply.** `assisted`. Rewriting a literal in another radix does not change its value but is a readability judgement; replacing an approximation by the exact expression changes the value by the truncation error, so confirm that nothing depends on the old value.

**Example.**
```ada
type Legal_Hex_Address is range 16#0000# .. 16#FFFF#;
One_Third : constant := 1.0 / 3.0;    --  not 0.33333_33333
```

**Source.** Q&S95 §3.1.2.

---

## NAM-04 Abbreviations

**Rule.** Do not abbreviate where a shorter synonym exists (`Time_Of_Receipt`, not `Recd_Time` or `R_Time`). An abbreviation must save many characters, be unambiguous (`Temp`: temporary or temperature?), be well accepted in the application domain, and appear on the project's list of accepted abbreviations. Use `renames` to shorten a long fully qualified name used many times in one region rather than inventing an abbreviation.

**Why.** Abbreviations are ambiguous out of context. Very long names obscure structure in nested code, so prefer short, unabbreviated words.

**Detect.** `assisted`. Identifiers containing consonant-only fragments of 3+ letters (`Rcvd`, `Msg`, `Cnt`, `Ptr`, `Idx`, `Tmp`, `Buf`, `Len`, `Num`, `Val`, `Str`) not on the project list.

**Apply.** `assisted` rename. Ask for, or create, an abbreviation list before renaming at scale.

**Source.** Q&S95 §3.1.4, §8.1.2 (domain context).

---

## NAM-05 Self-documenting names

**Rule.** Choose names that need no comment. Use the application's own vocabulary, but not obscure jargon. Short names are correct where the context supplies the meaning (`Left` in a tree walker; `A`, `B`, `C`, `X` in a formula that recalls the textbook equation). Do not use the same name for different kinds of entity (an object and a child package, for example): it causes visibility clashes.

**Detect.** `assisted`. Single-letter or two-letter names outside loops and formulae; names such as `Data`, `Info`, `Value`, `Temp`, `Flag`, `Result` used for long-lived objects.

**Apply.** `assisted` rename.

**Source.** Q&S95 §3.2.1; Ada 95 Rationale §8.1 for the visibility clash.

---

## NAM-06 Subtype (type) names

**Rule.** Name types and subtypes with singular, general nouns describing one of the values: `Day`, not `Days` or `Day_Type`. Consider suffixes only for visible access types, subranges and array types (`_Range`, `_Map`, `_Capacity`, `_Access`); their purpose is to avoid hunting for synonyms. Do not add such suffixes to private types; give them a plain noun. Do not reuse the subtype names of predefined packages (`Integer`, `String`, `Count`, `Positive`).

**Why.** Reads like English with the object naming rule ("Today is a Day") and matches the predefined names, which are `Integer` and `Boolean`, not `Integer_Type`.

**Detect.** `auto`. `type\s+\w+_Type\b`, `type\s+\w+_T\b`, plural type names (`type\s+\w+s\s+is`), redeclaration of predefined names: `(type|subtype)\s+(Integer|Natural|Positive|Float|Boolean|Character|String|Duration|Count)\s+is`.

**Apply.** `assisted` rename. `_Type` and `_T` suffixes are a very common project convention; if the project uses one consistently, report it once as a convention, do not rename.

**Example.**
```ada
type Day_Of_Month    is range      0 ..    31;
type Historical_Year is range -6_000 .. 2_500;
type Track_Map       is array (Sector_Range) of ...;
```

**Source.** Q&S95 §3.2.2; Q&S83 §3.2.2.

---

## NAM-07 Object names

**Rule.** Objects get singular, specific nouns describing the value during execution (`Today : Day`, `Retirement_Date : Date`). Boolean objects get predicate clauses or adjectives (`List_Is_Empty`, `Empty`, `Bright`). Record components get singular, general nouns, because the record object supplies the context (`Retirement_Date.Year`).

**Why.** Reads as prose: `if List_Is_Empty then`. If no single noun describes the object's value throughout its life, it is serving multiple purposes and should be split into several objects.

**Detect.** `assisted`. Boolean objects named without predicate form (`Flag`, `Status`, `Done_Flag`, `B`); plural names for scalar objects; objects reassigned with semantically different values in one scope.

**Apply.** `assisted` rename or split.

**Source.** Q&S95 §3.2.3.

---

## NAM-08 Tagged types and their packages

**Rule.** Pick one convention and apply it consistently. Option A (the guide's first instantiation): tagged types are named like any other type; packages exporting an abstraction with several implementations are prefixed `Abstract_`; mixin packages are suffixed `_Mixin`. Option B (Rosen 1995): the package is named after the class (`Shape`, `Line`), the main tagged type is `Instance`, a `subtype Class is Instance'Class` follows it, and mixin packages are suffixed `_Facet`. Option B deliberately forces full qualification (`Shape.Instance`) since every `Instance` hides every other.

**Why.** Consistency; the choice between highlighting inheritance (B) and treating it like any other abstraction (A) is a project decision. Changing later is expensive, as every name must change.

**Detect.** `assisted`. Mixed use of `Instance`/`Class` and ordinary type names across tagged-type packages. `_Type` suffixes on tagged types alongside plain names.

**Apply.** `advisory`. Report inconsistency; do not rename without a project decision.

**Source.** Q&S95 §3.2.4, §9.2.4, §9.5.1.

---

## NAM-09 Program unit names

**Rule.**
- Procedures and entries: action verbs (`Get_Next_Token`, `Create`).
- Boolean functions: predicate clauses (`Is_Empty`, `Is_Last_Item`).
- Other functions: nouns for what they return (`Successor`, `Length`, `Top`).
- Packages: noun phrases naming the abstraction, at a higher level than subprograms (`Terminals`, `Text_Routines`).
- Tasks: names implying an active entity (`Terminal_Resource_Manager`).
- Protected units: nouns for the data protected (`Current_Location`, `Guardian`).
- Generics: named as if they were not generic, but more general than their instantiations.

**Why.** Code reads as natural language: `if Stack.Is_Empty (Current_List) then Current_Token := Stack.Top (Current_List);`.

**Detect.** `assisted`. `function\s+(Get_|Set_|Do_|Compute_|Calculate_)` (verb-named functions, but note `Get_` is common and acceptable in many projects); `procedure\s+(\w+)` where the name is a noun; `function\s+\w+\s*\(.*\)\s*return\s+Boolean` where the name is not a predicate (`Check_`, `Valid`, `Empty_Status`).

**Apply.** `assisted` rename.

**Source.** Q&S95 §3.2.5; Q&S83 §3.2.4.

---

## NAM-10 Constants and named numbers instead of literals

**Rule.** Use symbolic values instead of literals wherever possible. Prefer a named number (`X : constant := 512;`, no type) when the value is truly universal and must be static; use a typed constant when the value belongs to a type. Use `Ada.Numerics.Pi` and `e`. Use `Ada.Characters.Latin_1` names rather than `Character'Val (7)`. Derive related values by static expression from a linearly independent set of literals (`Buffer_Size : constant := Pages_Per_Buffer * Bytes_Per_Page;`), so one literal change propagates. Use `'First`, `'Last`, `'Length`, `'Range` instead of literal bounds. Use `constant` for every object whose value cannot change after elaboration.

**Why.** Literals need comments; names do not. Static expressions are computed at compile time, so there is no cost. Named numbers get full compile-time checking. Stated exception: universally familiar, unchangeable values such as `32.0 + (9.0 * Celsius) / 5.0`.

**Detect.** `auto` candidates, `assisted` decision. Numeric literals other than `0`, `1`, `2` in statements (not in declarations): `:=.*\b[0-9]{2,}\b`, `(=|/=|<|>)\s*[0-9]{2,}`; character literal comparisons `= '\$'`; loops `for I in 1 .. 80`; `'Val\s*\(\s*[0-9]`; variables that are never assigned after declaration (`X : T := ...;` with no later `X :=`), which should be `constant`.

**Apply.** `assisted`. Introducing a constant is safe; choosing its name needs the reader. Making an object `constant` is safe when the compiler accepts it.

**Example.**
```ada
Bytes_Per_Page   : constant := 512;
Pages_Per_Buffer : constant := 10;
Buffer_Size      : constant := Pages_Per_Buffer * Bytes_Per_Page;   --  not 5_120
Area := Pi * Radius**2;                                              --  not 3.14159 * ...
```

**Source.** Q&S95 §3.2.6, §4.1.4 (constant versus parameterless function); Q&S83 §3.2.5.

---

## NAM-11 Exception names

**Rule.** Name exceptions for the kind of problem they report, meaningfully to the clients of the declaring package: `Invalid_Name`, `Stack_Overflow`. Not `Error`, `Failure`, `My_Exception`.

**Detect.** `auto`. `\b(Error|Failure|Exception|Err|Problem)\s*:\s*exception`, and names ending `_Exception`.

**Apply.** `assisted` rename.

**Source.** Q&S95 §3.2.7.

---

## NAM-12 Constructors

**Rule.** Name operations that create or initialise an object with a prefix such as `Make_`, `Create_` or `New_`; reserve `New_` for constructors returning an access value, since it suggests an allocator. If constructors are collected in a child package, name it `<Parent>.Constructor` (or similarly descriptive).

**Detect.** `assisted`. Functions returning a tagged or private type whose name is a bare noun identical to the type (`function Square return Square` is illegal anyway) or `Init`, `Initialize` returning a value.

**Apply.** `assisted` rename.

**Source.** Q&S95 §3.2.8, §9.3.3.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| NAM-01 | Underscores between words | assisted |
| NAM-02 | Radix appropriate to the problem; exact fractions as static expressions | assisted |
| NAM-04 | Abbreviations only from a project list; prefer short synonyms | assisted |
| NAM-05 | Self-documenting names; no name reuse across entity kinds | assisted |
| NAM-06 | Singular general nouns for types; no `_Type` unless project convention; no predefined names | assisted |
| NAM-07 | Specific nouns for objects, predicates for Booleans, general nouns for components | assisted |
| NAM-08 | One consistent convention for tagged types and class packages | advisory |
| NAM-09 | Verbs for procedures, predicates for Boolean functions, nouns for functions and packages | assisted |
| NAM-10 | Symbolic values and named numbers instead of literals; `constant` where possible | assisted |
| NAM-11 | Exceptions named for the problem | assisted |
| NAM-12 | `Make_`/`Create_`/`New_` for constructors | assisted |
