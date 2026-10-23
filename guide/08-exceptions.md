# 8. Exceptions

Rules `EXC-*`. Source: Ada 95 Quality and Style (Q&S95) §4.3 (exceptions in an abstraction's interface), §5.8 (handling), §7.5 (portability); Q&S83 §4.3, §5.8.

Part A covers the interface: which exceptions a unit declares and raises. Part B covers handling inside bodies. Changes here alter error behaviour, so nothing in this file is `auto`.

---

## Part A: Exceptions as part of an abstraction

## EXC-01 Declare and export the exceptions a unit can raise

**Rule.** Declare, in the visible part of the package, every exception the unit can raise, so clients can name them in handlers rather than falling back to `when others`. Do not borrow an exception declared elsewhere for a different purpose. Document, per subprogram and entry, which exceptions it can raise ([COM-03](03-comments.md)); describe each exception's meaning in general terms next to its declaration ([COM-05](03-comments.md)).

**Why.** Clients cannot handle what they cannot name; reusing another package's exception couples the two packages and blurs the meaning.

**Detect.** `auto`. Exceptions declared in a package body and raised in a visible subprogram; `raise Other_Package.Some_Error` from a package that is not that package's child.

**Apply.** `assisted`. Move the declaration to the spec; for a borrowed exception, declare a new one and convert.

**Source.** Q&S95 §4.3.1.

---

## EXC-02 One exception per distinguishable error; one for the irrecoverable rest

**Rule.** Do not raise the same exception for different errors that a client could tell apart and act on differently. For internal errors the client cannot recover from, export a single exception (an `Internal_Error`) and distinguish the causes inside the unit, attaching detail with `Ada.Exceptions` (`raise X with "..."` in Ada 2005 and later). That gives clients N + 1 exceptions: N recoverable, one not.

**Detect.** `assisted`. A package exporting one `Error` exception raised from many places with no message; or conversely a dozen exceptions that every client handles identically.

**Apply.** `advisory`.

**Source.** Q&S95 §4.3.1.

---

## EXC-03 Do not raise for what the unit can avoid or correct; provide interrogatives

**Rule.** Raise only for conditions the abstraction cannot prevent or correct itself; never for status or modes incidental to completing an operation. Offer query functions (`Is_Empty`, `Is_Full`, `Exists`) so a client can avoid triggering the exception at all.

**Detect.** `assisted`. Exceptions used as the normal "not found" return; packages with `Underflow`/`Overflow`-style exceptions but no `Is_Empty`/`Is_Full` queries; handlers that catch and then continue as the normal path.

**Apply.** `assisted`. Add the interrogative function; change callers that use exception handlers for control flow.

**Source.** Q&S95 §4.3.1.

---

## EXC-04 Leave state unchanged when raising

**Rule.** When an operation cannot complete, detect that before changing internal state, or restore the state before raising, so the object remains usable afterwards. The guide notes this is always desirable but not always possible.

**Detect.** `assisted`. Bodies that modify fields or counters and then `raise` in a later branch without an undo.

**Apply.** `assisted`. Reorder checks before mutations.

**Source.** Q&S95 §4.3.1.

---

## EXC-05 Do not raise predefined exceptions; convert the ones you catch

**Rule.** Never explicitly `raise Constraint_Error`, `Program_Error`, `Storage_Error` or an implementation-defined exception; raise a user-defined one that names the problem. Catch predefined and compiler-defined exceptions at the earliest opportunity and convert them (raise a user-defined exception in the handler) into terms of the abstraction. Never let an exception propagate beyond the scope in which it is declared: the client, who does not know about the units used inside your body, could handle it only with `when others`.

**Detect.** `auto`. `raise\s+(Constraint_Error|Program_Error|Storage_Error|Tasking_Error|Numeric_Error)\b`; `raise\s+(System\.|GNAT\.|Ada\.IO_Exceptions\.)` outside wrappers; exceptions declared in a body or a declare block that can escape it.

**Apply.** `assisted`. Introduce a named exception, convert in the handler. Note that GNAT and some projects deliberately `raise Program_Error with "unreachable"`; treat that as a project convention if consistent. A `raise Constraint_Error` used to signal a contract violation is better expressed as a `Pre` aspect in Ada 2012 and later ([14-modern-ada.md](14-modern-ada.md)).

**Example.**
```ada
exception
   when Ada.IO_Exceptions.Name_Error =>
      raise Configuration_Missing with "no file " & Path;
```

**Source.** Q&S95 §4.3.1.

---

## Part B: Handling exceptions

## EXC-06 Avoid the exception when a cheap check exists; otherwise handle it

**Rule.** When a condition can be tested easily and reliably as part of the algorithm (null pointer before dereference, zero before division, `Is_Empty` before `Pop`), test it instead of handling the exception. When the test would duplicate the work (searching a list to avoid `Entry_Not_Found` before searching it to update), would be unreliable (file exists, then is deleted before the open), or is harder than the operation (numeric overflow), do the operation inside a handler. Keep handlers for fault handling only, so the reader can skip them and still follow the normal flow. Never raise and handle an exception within the same unit as a way to exit a loop, `if`, `case` or block: that is a `goto`.

**Detect.** `auto`. `raise` and a handler for the same exception in one body; `when Constraint_Error => null;` wrapping a conversion that could be guarded with `'Valid` or a range test; handlers that assign a result and continue as the normal path.

**Apply.** `assisted`.

**Source.** Q&S95 §5.8.1.

---

## EXC-07 `others` handlers: last resort, and record what was caught

**Rule.** Use `when others` only for exceptions that cannot be enumerated, preferably to flag an impending abort, in the frame of a task or the main subprogram where final clean-up and orderly shutdown belong. Always name the occurrence (`when E : others =>`) and log `Ada.Exceptions.Exception_Name`, `Exception_Message` or `Exception_Information` before converting or re-raising, otherwise the original raise point is lost. During development, let `others` capture what it catches and then add an explicit handler for that exception. Do not use `Exception_Id` to discriminate in an `others` handler; it is implementation-defined. Do not evaluate an object that became abnormal through the failed check that raised the exception.

**Detect.** `auto`. `when others\s*=>\s*null;` (swallowed exception); `when others =>` with no `E :` occurrence name and no `Exception_` call; `when others => raise;` deep inside a body (acceptable at a frame boundary).

**Apply.** `auto` for adding the occurrence name and a logging call matching the project's logging convention; `assisted` for replacing `others` with explicit handlers.

**Example.**
```ada
exception
   when Out_Of_Bounds : Constraint_Error =>
      Put_Line ("Input choice not in range: " & Exception_Name (Out_Of_Bounds));
   when The_Error : others =>
      Put_Line ("Unexpected error: " & Exception_Information (The_Error));
      raise;
```

**Source.** Q&S95 §5.8.2, §5.8.1 (abnormal objects); ARM §13.9.1.

---

## EXC-08 Handle everything somewhere; propagate only from the wrong level

**Rule.** Every exception that can be raised, user-defined or predefined, must have a handler in some frame that stops it leaving the abstraction unhandled. "It can never happen" is not a design. Catch and propagate (or substitute) only when the handler is at the wrong abstraction level to recover. An explicit decision to terminate (after cleanup: close files, release devices, raise alarms) is control; an unhandled exception is not.

**Detect.** `auto`. Main subprograms and task bodies with no handler; library-level packages whose visible subprograms can propagate predefined exceptions with no mention in the spec header.

**Apply.** `assisted`.

**Source.** Q&S95 §5.8.3.

---

## EXC-09 Localise the cause with small blocks and messages

**Rule.** Do not rely on being able to tell which statement raised a predefined or implementation-defined exception; convert those to the abstraction's exceptions and propagate. For user-defined exceptions, attach handlers to small blocks ([STM-17](06-statements.md)) so the cause is narrowed and the subprogram can continue after recovery. Attach a message when raising (`raise X with "..."` in Ada 2005 and later; `Ada.Exceptions.Raise_Exception` before). Granularity is a judgement: too small and the error handling dwarfs the algorithm, too large and the cause is lost.

**Detect.** `assisted`. Subprogram-level handlers covering many statements that can raise the same exception; `raise X;` with no message where several raise points share `X`.

**Apply.** `assisted`. Adding messages is safe.

**Source.** Q&S95 §5.8.4.

---

## EXC-10 Do not suppress checks during development; suppress narrowly in production

**Rule.** Do not use `pragma Suppress` (or `-gnatp`) during development. If profiling shows that checks must go in production, suppress them in the smallest block of statements that can safely do without them, with a comment, leaving checks on around it. A suppressed check that would have failed makes execution erroneous, and exceptions from called units may still arrive.

**Detect.** `auto`. `pragma Suppress` at library or unit level; `Suppress (All_Checks)`.

**Apply.** `assisted`. Narrow the scope to a block.

**Source.** Q&S95 §5.9.5.

---

## EXC-11 No exceptions out of Finalize, Adjust or Initialize

**Rule.** Give every user-defined `Finalize` and `Adjust` a handler for all exceptions at the end of the procedure. An exception propagating out of them is a bounded error: it is ignored or becomes `Program_Error`. Treat `Initialize` the same way.

**Detect.** `auto`. `procedure (Finalize|Adjust|Initialize) \(.*in out` bodies with no `exception` section.

**Apply.** `auto`. Add `exception when others => null;` only if the project agrees swallowing is right; otherwise log then swallow. Record the decision in a comment.

**Source.** Q&S95 §5.9.8; ARM §7.6.1.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| EXC-01 | Declare and export every exception a unit raises; document per subprogram | assisted |
| EXC-02 | One exception per distinguishable error, one for the irrecoverable rest | advisory |
| EXC-03 | Raise only for what the unit cannot avoid; provide interrogatives | assisted |
| EXC-04 | Leave state unchanged when raising | assisted |
| EXC-05 | Never raise predefined exceptions; convert what you catch; nothing escapes its scope | assisted |
| EXC-06 | Cheap checks instead of handlers; handlers only for faults; no exception-as-goto | assisted |
| EXC-07 | `others` only at frame boundaries, named and logged | auto/assisted |
| EXC-08 | Every exception handled somewhere; propagate only from the wrong level | assisted |
| EXC-09 | Small handler blocks; messages on raise | assisted |
| EXC-10 | No check suppression in development; narrow in production | assisted |
| EXC-11 | Handlers in `Finalize`, `Adjust`, `Initialize` | auto |
