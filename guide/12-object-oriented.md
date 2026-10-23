# 12. Object-Oriented Features

Rules `OOP-*`. Source: Ada 95 Quality and Style (Q&S95) chapter 9; Ada 95 Rationale §4; Taft 1994/1995; Barnes 1996. Q&S83 has no counterpart. For Ada 2005 additions that change these idioms (interfaces, `overriding` indicators, `Object.Operation` notation, anonymous access-to-class-wide parameters), see [14-modern-ada.md](14-modern-ada.md).

Inheritance in Ada is for "is-a" hierarchies that come out of a design, not for module inclusion (`with`/`use` do that) and not for "has-a" (record composition does that). Most rules here are design-level and `advisory`; the ones about `Initialize`/`Finalize` chaining, equality on extensions, and constructors as primitives are mechanical enough to detect and often to fix.

---

## OOP-01 Type extension for is-a hierarchies; one tagged type per package; a class-wide access type

**Rule.** Use tagged types and type extension to implement a generalisation/specialisation hierarchy, and to keep one interface across several implementations. Define one tagged type per package (exceptions: two types that need mutual visibility and are always used together, or a small all-abstract hierarchy, with the maintainability cost acknowledged, and child packages preferred). Alongside the type, declare a general access-to-class-wide type (`type Reference is access all Object'Class;`) for clients that build heterogeneous structures, unless the type will never be in a collection. If a descendant would have to remove a component or operation of its ancestor, extension is the wrong tool.

**Detect.** `auto`. Package specs declaring several tagged types; tagged types with no corresponding `access all T'Class` anywhere while clients declare their own; derivations whose overriding operations raise `Program_Error`/"not supported" (removal by stealth).

**Apply.** `advisory`. Splitting a package with several tagged types into children is a safe but client-visible restructuring.

**Source.** Q&S95 §9.2.1, §9.4.1; §5.4.3.

---

## OOP-02 Substitutability: weaker preconditions, stronger postconditions, same exceptions

**Rule.** Every dispatching operation of every type in a class rooted at `T` must satisfy the semantics clients expect of the operation on `T'Class`: the same or weaker precondition, the same or stronger postcondition, and no exceptions the users of the overridden operation do not know about. If a derived type needs a stronger precondition, either introduce a dispatching query (`Is_Valid`) that clients check first, or make failure part of the class-wide contract by raising one documented exception from every implementation. (In Ada 2012 and later, `Pre'Class` and `Post'Class` express this and the compiler enforces the weakening/strengthening, see [14-modern-ada.md](14-modern-ada.md).) Exception: a type extension used purely as an implementation technique for a non-tagged private type need not preserve the class-wide semantics, because clients never see it as a member of the class.

**Why.** Clients of `T'Class` must be able to reason about the operations without reading every implementation, and adding a type must not invalidate existing class-wide code.

**Detect.** `assisted`. Overriding bodies that start with a guard raising an exception the parent's spec does not mention; overriding operations whose header comments add "must" clauses; overrides that do nothing (`null;`).

**Apply.** `advisory`; `assisted` for adding the missing exception documentation or an `Is_Valid` query.

**Source.** Q&S95 §9.2.2, §9.3.1; Wegner and Zdonik 1988.

---

## OOP-03 Controlled types for resources; chain Initialize and Finalize to the parent

**Rule.** Any type that allocates resources needing release on destruction or overwrite is derived from `Ada.Finalization.Controlled` or `Limited_Controlled`, instead of exporting a `Cleanup`/`Destroy` the client has to remember to call. `Finalize` undoes what `Adjust` did (deep copy on adjust, deallocate on finalize). A derived type's `Initialize` calls the parent's `Initialize` (normally first) and its `Finalize` calls the parent's `Finalize` (normally last). Consider making the resource-holding *component* controlled rather than the enclosing structure, so the enclosing type need not be tagged.

**Detect.** `auto`. Packages exporting `Destroy`, `Free`, `Cleanup`, `Release`, `Close` for a private type that is not controlled; overriding `Initialize`/`Finalize`/`Adjust` bodies that do not call the parent's (`Parent_Type (Object)` conversion followed by the call); types with an `Adjust` but no `Finalize`.

**Apply.** `auto` for adding the parent call to an override (safe when the parent's operation is not `null`; check); `assisted` for converting to a controlled type.

**Example.**
```ada
overriding procedure Finalize (List : in out Linked_List) is
begin
   Release_All_Nodes (List);
   Parent_Type (List).Finalize;   --  or Finalize (Parent_Type (List));
end Finalize;
```

**Source.** Q&S95 §9.2.3, §5.4.5, §8.3.1; ARM §7.6.

---

## OOP-04 Abstract types for roots, internal nodes and alternative implementations

**Rule.** Declare the root and the internal nodes of a classification hierarchy `abstract` when only the leaves are meaningful objects; declare the common operations abstract so each concrete derivation must provide them. Use an abstract type with abstract operations as the single interface behind several implementations of one abstraction (which, unlike multiple package bodies for one spec, lets several implementations coexist in one program), and as the formal derived type of a generic.

**Detect.** `auto`. Tagged root types that are not abstract but have no objects declared anywhere and whose primitive bodies are stubs (`null;`, `raise Program_Error`); hierarchies where the intermediate types' operations are all overridden by every leaf.

**Apply.** `assisted`. Making a type abstract is compile-checked: any object declarations or non-overridden abstract operations are reported.

**Source.** Q&S95 §9.2.4; Ada 95 Rationale §4.2, §4.4.3.

---

## OOP-05 Primitive abstract when there is no default; document redispatching

**Rule.** Make a primitive operation abstract when there is no meaningful default behaviour for the class (area of a shape); make it concrete when there is one. Remember that everything declared after the type in the same package and before the next type declaration is a primitive operation and will be inherited; operations that must not be inherited are class-wide operations or live in a child package. When an operation's implementation redispatches (`Width (Any_Rectangle'Class (R))`) with the intent that derived types override the redispatched-to operation, document that intent in the specification as part of the type's interface to its extenders; and mark every use of redispatching in the body in a project-consistent way so maintainers can find it. Undocumented redispatching is the classic inheritance maintenance trap: overriding one operation silently changes another several levels up.

**Detect.** `auto`. `T'Class (X)` conversions inside primitive operation bodies with no comment mentioning dispatch; specs of types whose bodies redispatch but whose headers do not say which operations are hooks.

**Apply.** `auto` for adding a `--  Redispatches to Width/Height` marker comment at the redispatch; `assisted` for the spec documentation.

**Source.** Q&S95 §9.3.1; Volan 1994.

---

## OOP-06 Class-wide operations for algorithms over the whole class, and for what must not be inherited

**Rule.** Write an operation with a parameter of `T'Class` when it can be written, compiled and tested without knowing all descendants (`Moment (OC : Object'Class)` calling the dispatching `Area`), and when it must not be inherited or overridden. Do not redefine a class-wide operation for a derived class; the ambiguity is confusing.

**Detect.** `auto`. Primitive operations whose bodies only call other dispatching operations and never touch the type's components (candidates for class-wide); operations declared with `T'Class` parameters redeclared with `Derived'Class` parameters of the same name.

**Apply.** `assisted`. Changing a primitive to class-wide removes it from the inheritance set, so check that no derived type overrides it.

**Source.** Q&S95 §9.3.2; Barnes 1996.

---

## OOP-07 Constructors: not primitive abstract; in a child package; split from initialisation

**Rule.** A constructor (any operation returning an initialised instance) should not be a primitive abstract operation, and should be primitive at all only if no derived type will need extra parameters, because inherited constructors cannot add parameters and so cannot construct descendants (and a function returning the type becomes abstract automatically on extension to an abstract type). Declare constructors in a child package (`Game.Constructors`) or nested package so they are not inherited and can change without disturbing the parent's clients. Split construction from initialisation (`Initialize (Self, Make)` callable by the derived type's `Initialize`, then `Create` allocates and calls it). For limited types, constructors return an access value or initialise through an access discriminant; do not rely on default initialisation for a tagged type. Name constructors per [NAM-12](02-naming.md).

**Detect.** `auto`. `function \w+ \(.*\) return T is abstract;` where `T` is the tagged type; `function Create (...) return T` declared as a primitive in a package whose type has extensions with extra components; `Create`/`Make` returning tagged types without a child package.

**Apply.** `assisted`. Moving a constructor into a child package changes callers' `with` clauses and names.

**Source.** Q&S95 §9.3.3, §3.2.8; Dewar 1995; Taft 1995b. (Ada 2005 extended return statements and Ada 2012 build-in-place functions for limited types change the "return an access" advice; see [14-modern-ada.md](14-modern-ada.md).)

---

## OOP-08 Equality on tagged types: check the extensions

**Rule.** When `"="` is redefined on a tagged type, the inherited equality on an extension compares the parent part with the redefined `"="` and the extension components with predefined equality. Decide for each extension whether that is right (two circles whose centres are "near" but whose radii are compared exactly is probably not) and override `"="` on the extension, calling the parent's on the parent part.

**Detect.** `auto`. A tagged type with `function "="` and extensions of it with components and no `"="` of their own.

**Apply.** `assisted`.

**Source.** Q&S95 §9.3.4, §5.7.5; Barnes 1996.

---

## OOP-09 Class-wide programming versus generics

**Rule.** Use class-wide programming (tagged types, class-wide types, dispatching) for large, reusable, extensible frameworks, for a consistent interface across a family of types, and in place of variant records. Use generics for small reusable algorithms and idioms (sorts, maps, bags, iterators) and to define a type in terms of another type as a container or composite: `generic type Element is private; package Stack` is preferable to a `Stack` package exporting a tagged `Element` root that every element type must descend from. Avoid type extension as a substitute for generic parameterisation; the inheritance-based form can always be recovered by instantiation.

**Detect.** `auto`. Container packages exporting `type Element is tagged null record;` or `type Item is abstract tagged ...` as the element base; many small generics with large bodies instantiated many times (consider a class-wide design).

**Apply.** `advisory`.

**Source.** Q&S95 §9.3.5, §8.3.2; §5.3.2, §5.4.7.

---

## OOP-10 Derived types see the parent as clients do, unless declared in a child

**Rule.** By default give a derived tagged type the same visibility of its parent as any other client, which decouples it from the parent's implementation. When the extension genuinely needs the parent's private representation (a bounded stack updating the abstract stack's `Size`), declare it in a child package of the parent rather than adding public operations that expose the representation to everyone.

**Detect.** `auto`. Packages adding `Get_Internal_X`/`Set_Internal_X` operations to a tagged type's spec that are used only by derived types; derived types in unrelated packages using `Unchecked_Conversion` or `Unchecked_Access` to reach the parent's components.

**Apply.** `assisted`. Moving the derived type into a child package changes its name and its clients' `with` clauses.

**Source.** Q&S95 §9.4.1; §8.4.8; Barbey, Kempe and Strohmeier 1994.

---

## OOP-11 Multiple inheritance by composition, mixins and access discriminants

**Rule.** Ada has no multiple-inheritance syntax; it has building blocks. For inheriting an *implementation*, compose: a record component of the implementing type, with glue code. For adding optional *features* to a core abstraction, use generic mixins (`generic type Some_Window is new Window with private; package Label_Mixin is type Window_With_Label is new Some_Window with private; ...`) and chain the instantiations in the private part of the final package. For "full" multiple inheritance, where one object must be usable as a member of two unrelated classes, use access discriminants (`type Savings (Self : access Object'Class) is new Savings_Account.Object with null record;`) with components `As_Savings : Savings (Object'Access)` whose operations call through to the enclosing object. Do not use inheritance to derive an unrelated abstraction from two parents (a menu from a command line and a window), and never for parts-of relations, which are record composition ([TYP-08](05-types.md)). Where two parents share a common ancestor's state, use an abstract hierarchy for the interfaces and mixins for the implementation to avoid inheriting the state twice. (Ada 2005 interfaces give multiple inheritance of *interface* directly, see [14-modern-ada.md](14-modern-ada.md).)

**Detect.** `assisted`. Types derived from an abstraction they do not "is-a"; deep single-inheritance chains that exist only to accumulate unrelated features.

**Apply.** `advisory`.

**Source.** Q&S95 §9.5.1, §8.3.8; Taft 1994; Ada 95 Rationale §4.6.2, §4.6.3.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| OOP-01 | Extension for is-a; one tagged type per package; class-wide access type | advisory |
| OOP-02 | Substitutability: weaker pre, stronger post, same exceptions | advisory |
| OOP-03 | Controlled types for resources; chain `Initialize`/`Finalize` to parent | auto/assisted |
| OOP-04 | Abstract roots, internal nodes, and interfaces to multiple implementations | assisted |
| OOP-05 | Abstract primitives without a default; document redispatching | auto/assisted |
| OOP-06 | Class-wide operations for whole-class algorithms and non-inherited operations | assisted |
| OOP-07 | Constructors not primitive abstract; in child packages; split from `Initialize` | assisted |
| OOP-08 | Override `"="` on extensions of types with redefined equality | assisted |
| OOP-09 | Class-wide programming for frameworks, generics for small algorithms and containers | advisory |
| OOP-10 | Derived types in a child package when they need the parent's representation | assisted |
| OOP-11 | Multiple inheritance via composition, mixins and access discriminants | advisory |
