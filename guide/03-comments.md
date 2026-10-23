# 3. Comments

Rules `COM-*`. Source: Ada 95 Quality and Style (Q&S95) §3.3; Q&S83 §3.3.

The guides' position: readability comes from naming and structure, not from comment volume. Comments exist to say what cannot be said in Ada, to state intent, to mark structure, and to flag deliberate deviations. A comment that paraphrases the code is a liability because nothing keeps it in step with the code. When reviewing, removing a bad comment is as valid a finding as adding a missing one.

Comment-text changes never alter program behaviour, so every rule here is safe to apply once the content has been judged.

---

## COM-01 General comment discipline

**Rule.** Make the code clear enough to need few comments. Never repeat in a comment what the code already says. Where a comment is needed, make it concise, complete, grammatical and correctly spelled. Make comments visually distinct from code (indented with the code, grouped into headers, boxed with dashed lines). Structure header comments so a tool can extract them.

**Why.** Duplicated information is work to write, drifts out of date at the first change, and hides the few comments that matter. Automatically generated stale comments are worse than hand-written ones because readers trust them.

**Detect.** `assisted`. Comment lines whose words largely repeat the identifiers on the next code line; comments ending in a different column from the code they annotate; comment density above roughly one comment line per code line in a body.

**Apply.** `assisted`. Delete paraphrasing comments; fix spelling; keep comments that explain intent.

**Source.** Q&S95 §3.3, §3.3.1.

---

## COM-02 File header

**Rule.** Put a header at the top of every source file carrying ownership (copyright notice), responsibility (author and department or team, not phone numbers or office locations, which go stale) and, if the configuration management tool does not provide it, a revision history. The guide's own note: with modern version control the revision history in the header is superfluous and less reliable than the tool's; prefer the tool.

**Detect.** `auto`. First non-blank line of a `.ads`/`.adb` file is not a comment, or the first comment block lacks a copyright or licence line.

**Apply.** `auto` for inserting a header template matching the project's existing headers; `assisted` for choosing its content. Do not add revision-history blocks to a project under version control.

**Source.** Q&S95 §3.3.2.

---

## COM-03 Specification header: what the unit does, for its user

**Rule.** Each program unit specification gets a header with everything a user needs that the Ada specification itself does not state: the unit's purpose; how the visible subprograms of a package interact and are meant to be used together (usage protocol); every exception the unit can raise; every global effect; preconditions and postconditions (in Ada 2012 and later, prefer `Pre`/`Post` aspects, see [14-modern-ada.md](14-modern-ada.md)); hidden tasks the unit activates; time and space characteristics. Do not list parameter names, modes or subtypes; do not list the package's subprograms just to list them; do not list the units it uses or is used by; do not describe how the unit is implemented or how it fits into the enclosing system ("this unit is called by X").

**Why.** A user should be able to use the unit from its specification and header without reading the body. Duplicated interface detail goes stale. Implementation detail in the spec invites clients to depend on it. Resource usage, especially tasks that keep running after a call returns, is information a user cannot get any other way.

**Detect.** `assisted`. Visible subprograms with no comment at all; headers that enumerate parameters (`-- Parameters:` followed by names already in the profile); headers naming callers; packages exporting exceptions with no statement of which operations raise them.

**Apply.** `assisted`. Write or trim the header. A single header may cover a closely related group (`Max`/`Min`, `Sin`/`Cos`/`Tan`), especially when they raise the same exceptions.

**Example.**
```ada
---------------------------------------------------------------------
--  Position_Of
--  Purpose:
--    Returns the coordinate position of the specified node. The
--    default position (0,0) is returned if no position has been
--    assigned yet.
--  Exceptions:
--    Node_Not_Defined
---------------------------------------------------------------------
function Position_Of (Current : in Node) return Position;
```

**Source.** Q&S95 §3.3.3.

---

## COM-04 Body header: how and why, for the maintainer

**Rule.** The body header carries what the maintainer needs: portability issues; a summary of a complex algorithm (not a paraphrase of the code); reasons for significant or controversial implementation decisions; alternatives tried and discarded, with the reason; anticipated changes, especially where the code is already prepared for them. Do not repeat what the specification header says or what the code shows. If there is nothing to say, omit the body header entirely rather than writing "Implementation Notes: None". Be aware that "Non-portable features: None" is a guarantee, not a shrug.

**Why.** The most valuable comment to a maintainer is a clear statement of why a change they are about to make will not work.

**Detect.** `assisted`. Body headers that duplicate the spec header; headers consisting only of "None" entries; bodies containing `pragma Import`, address clauses, `Unchecked_Conversion` or `System` dependencies with no portability note.

**Apply.** `assisted`.

**Source.** Q&S95 §3.3.4.

---

## COM-05 Data comments

**Rule.** Comment every type, object and exception whose name is not self-explanatory. Describe the semantic structure of pointer-based structures (which lists are sorted, which are doubly linked, and why). Describe invariants and relationships maintained between objects (in Ada 2012 and later, consider `Type_Invariant` or a predicate instead). State the conditions under which each exception is raised; grouping a package's exceptions under one heading gives the reader a glossary. Describe exceptions in general terms, not as a list of the subprograms that raise them (that list is kept per subprogram and goes stale here). Omit comments that merely repeat the name. For tagged types, document redispatching operations that derived types are expected to override.

**Why.** Maintainers read the data structures first. One good comment on a declaration replaces many comments at the use sites.

**Detect.** `auto` for undocumented exception declarations in a package spec (`\w+\s*:\s*exception\s*;` with no adjacent comment) and for record types with several access components and no comment block. `assisted` otherwise.

**Apply.** `assisted`.

**Example.**
```ada
Node_Already_Defined : exception;   --  Raised when an attempt is made to
                                    --  define a node with an identifier
                                    --  which already defines a node.
```

**Source.** Q&S95 §3.3.5, §9.3.1.

---

## COM-06 Statement comments

**Rule.** Minimise comments among statements. Comment only what is not obvious: why a construct was chosen, a compiler-bug workaround, non-portable or environment-dependent code, a deliberately lower-level solution and the higher-level attempt that failed. Comment intentional omissions ("not deallocated here because ...") so a maintainer does not "fix" them. Never paraphrase the code; never describe what another unit does internally (that violates information hiding and goes stale). Make the remaining comments visually distinct.

**Detect.** `assisted`. Comments immediately preceding a statement that restate it (`-- Increment count` before `Count := Count + 1`); comments describing the called subprogram's algorithm; a comment on nearly every line.

**Apply.** `assisted`. Delete paraphrases, keep intent. The guide's own example shrinks six comments to one and names the loop instead.

**Example.**
```ada
Sum_Integers_Converted_From_Strings :
   for I in Strings'Range loop
      Number := Convert_To_Integer (Strings (I));
      Sum    := Sum + Number;
      Count  := Count + 1;
      --  The global Max_Number is computed here for efficiency.
      if Number > Max_Number then
         Max_Number := Number;
      end if;
   end loop Sum_Integers_Converted_From_Strings;
```

**Source.** Q&S95 §3.3.6.

---

## COM-07 Marker comments

**Rule.** Use the dashed-line pagination markers of [LAY-07](01-layout.md#lay-07-pagination-mark-unit-boundaries). Repeat the unit name after `begin` when the `begin` of a package body, subprogram body, task body or block is far from its first line because of declarations (`begin -- Abstract_Strings`). For long or deeply nested `if` and `case` statements, mark `end if;` / `end case;` with a comment summarising the condition, and mark a distant `else` with a comment stating what is true there. Do not do this for loops and blocks: name them instead, so the compiler checks the matching `end loop Name;` / `end Name;`.

**Why.** Marker comments resolve "where am I" over a visual distance, particularly across page breaks. Overdone, they are clutter: visual distance is the criterion.

**Detect.** `auto`. `if` statements spanning more than about 30 lines whose `end if;` has no trailing comment; `begin` more than about 20 lines below `package body`/`procedure` with no comment; unnamed loops longer than a screen. `assisted` for the threshold.

**Apply.** `auto`. Adding `-- Name` after `begin` or `end if;` is safe. Converting an unnamed long loop to a named one is safe (add the name before the loop and after `end loop`).

**Source.** Q&S95 §3.3.7.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| COM-01 | No paraphrase; concise, correct, visually distinct | assisted |
| COM-02 | File header with ownership and responsibility; history belongs to version control | auto |
| COM-03 | Spec header: purpose, protocol, exceptions, global effects, performance; no parameter lists | assisted |
| COM-04 | Body header: how and why, portability, decisions, discarded alternatives; omit if empty | assisted |
| COM-05 | Document data structures, invariants and exception conditions | assisted |
| COM-06 | Few statement comments; explain intent and omissions, never the code | assisted |
| COM-07 | Markers on distant `begin`, `else`, `end if`; name loops and blocks instead of commenting them | auto |
