# 9. Concurrency

Rules `CON-*`. Source: Ada 95 Quality and Style (Q&S95) chapter 6; §5.9.9, §5.9.10; Q&S83 chapter 6 (rendezvous-only era; its shared-variable and delay advice is superseded by protected objects and `delay until`).

The guide words most of these "consider", because concurrency decisions are design trade-offs. The reviewer's job is to name the trade-off and the rule; the few mechanical items (`when others` at the end of a task body, `delay until` for periodic loops, `Volatile`/`Atomic` on shared variables) can be applied directly. Everything else is `advisory` or `assisted` and must be tested under load after a change.

---

## CON-01 Protected objects for shared data and passive synchronisation

**Rule.** Use a protected object, not a task, for mutual exclusion, for data shared between tasks, and for passive synchronisation such as a resource monitor or a bounded buffer. Use a task only to introduce a new thread of control. Encapsulate the protected object in a package's private part or body behind a limited private type unless clients must use a `select` on its entries, in which case expose it. Keep protected operations short, bounded and non-blocking.

**Why.** Protected objects are the lightweight mechanism: no context switch, barriers re-evaluated at the end of each protected action, and far less code than a server task with a selective accept loop.

**Detect.** `auto`. A `task` whose body is a single `loop select accept ... or accept ... end select end loop` with no `delay` and no other activity (a passive server); packages exporting a task type whose only purpose is to guard data.

**Apply.** `advisory`. Rewrite as a protected type with procedures and entries; the entries' guards become barriers. Test under concurrency.

**Example.**
```ada
protected type Bounded_Buffer is
   entry Get (X : out Item);
   entry Put (X : in Item);
private
   Get_Index, Put_Index : Buffer_Index := 1;
   Count                : Buffer_Count := 0;
   Data                 : Buffer_Array;
end Bounded_Buffer;

protected body Bounded_Buffer is
   entry Get (X : out Item) when Count > 0 is ...
   entry Put (X : in Item) when Count < Maximum_Buffer_Size is ...
end Bounded_Buffer;
```

**Source.** Q&S95 §6.1.1, §6.2.1, §6.2.5; Ada 95 Rationale §II.9, §9.1.

---

## CON-02 Barriers: no globals, no side effects; no blocking inside protected operations

**Rule.** An entry barrier refers only to the protected object's own components (and `'Count` of its own entries). It must not read global variables, which can change without re-evaluating the barrier, and must not have side effects. Inside a protected entry, procedure or function never do anything potentially blocking: no `select`, `accept`, entry call, `delay`, `abort`, task creation, call on another protected operation of the same object, or call on a subprogram that does any of these, nor on operating-system primitives that may block. Doing so is a bounded error (`Program_Error`) or a deadlock.

**Detect.** `auto`. `when` barrier expressions naming identifiers not declared in the protected type's private part; `delay`, `accept`, `select`, `.Entry_Name (` calls, `new Task_Type`, `Ada.Text_IO` calls inside `protected body`. GNAT: `pragma Detect_Blocking`.

**Apply.** `assisted`. Move the global into the protected object (set it through a protected procedure); move blocking calls outside the protected action.

**Source.** Q&S95 §6.1.1, §5.9.9, §6.2.4; ARM §9.5.1.

---

## CON-03 Interrupt handlers as protected procedures with an adequate ceiling

**Rule.** Implement interrupt handlers as protected procedures attached with `Attach_Handler`/`Interrupt_Handler`. The protected object's ceiling priority must be at least the maximum priority of the interrupt; a higher-priority interrupt delivered to a lower-ceiling handler is erroneous under ceiling locking. Signal normal tasks from the handler by changing a component that opens a barrier.

**Detect.** `auto`. `Attach_Handler` on a protected object with no `pragma Interrupt_Priority` / `Interrupt_Priority` aspect; address clauses on task entries (obsolescent, [TYP-16](05-types.md)).

**Apply.** `assisted`.

**Source.** Q&S95 §6.1.1; Ada 95 Rationale §9.1, §C.3.2; ARM §C.3.1.

---

## CON-04 Tasks for genuine threads of control

**Rule.** Use a task to model an asynchronous entity in the problem domain, a cyclic activity (poll a device every 30 ms), a high-priority emergency activity, or a concurrent algorithm on a multiprocessor. Use the rendezvous only for genuinely synchronous, unbuffered, tightly coupled communication or time-ordered synchronisation; for data exchange use a protected object. Passive tasks should become protected objects ([CON-01](#con-01-protected-objects-for-shared-data-and-passive-synchronisation)).

**Detect.** `assisted`. Tasks whose only job is to hold data; tasks on a single processor that parallelise a pure computation (context-switch overhead may exceed the gain).

**Apply.** `advisory`.

**Source.** Q&S95 §6.1.2, §4.1.9.

---

## CON-05 Discriminants, not an initial rendezvous, to parameterise tasks and protected objects

**Rule.** Give a task type or protected type discriminants for its identity, priority, storage size, entry-family size, interrupt, and (through an access discriminant) the data it works on. Do not use an `Initialize` entry that every task must be called on before it can start; that is the Ada 83 idiom. Beware that data reached through an access discriminant can change later; copy it if the task must not see changes.

**Detect.** `auto`. Task types whose body begins with `accept Initialize (...)` or `accept Start (...)` storing parameters into task-local variables; tasks indexed by an array position that discover their own index through a rendezvous.

**Apply.** `assisted`. Replace the entry with discriminants and update each object declaration; callers' initial entry calls are deleted.

**Example.**
```ada
task type Producer (Channel : Channel_Number; ID : ID_Number);
Keyboard : Producer (Channel => Keyboard_Channel, ID => 1);
```

**Source.** Q&S95 §6.1.3; Ada 95 Rationale §9.1, §9.6, §II.9.

---

## CON-06 Single task and protected declarations for unique instances

**Rule.** Declare a task or protected object directly (`task Buffer;`, `protected Display is ...`) when there is exactly one; declare a type only when several identical objects are needed or when objects must be created dynamically or referenced by different names. Converting later is trivial, though introducing multiple instances may change the synchronisation network.

**Detect.** `auto`. `task type T` / `protected type T` with exactly one object declared of it in the project.

**Apply.** `assisted`.

**Source.** Q&S95 §6.1.4.

---

## CON-07 Minimise dynamic task creation; reuse workers

**Rule.** Task start-up is expensive. Instead of allocating a task per job, give tasks a top-level loop that waits for new work on an entry. If tasks must be dynamic (variable population, or a required activation order), treat them with the same discipline as any heap object: never overwrite the only access to a running task ([TYP-11](05-types.md)), and confirm `'Terminated` before `Unchecked_Deallocation` of a task object (freeing an unterminated discriminated task is a bounded error).

**Detect.** `auto`. `new Task_Type` inside a loop; a single access-to-task variable repeatedly assigned from `new`; `Unchecked_Deallocation` instantiated on an access-to-task type with no `'Terminated` test nearby.

**Apply.** `advisory`.

**Source.** Q&S95 §6.1.5, §6.3.2; ARM §13.11.2.

---

## CON-08 Priorities are not synchronisation

**Rule.** Do not rely on `pragma Priority` (or the `Priority` aspect) unless the compiler supports Annex D. Never use priorities to obtain a particular execution order or mutual exclusion; entry queues are FIFO by default, so a higher-priority caller is not served first (priority inversion), and the outcome of the race changes with any unrelated change or port. Use protected objects with ceiling locking (`pragma Locking_Policy (Ceiling_Locking)`) to minimise inversion.

**Detect.** `assisted`. Comments or code that assume ordering from priorities (`-- high priority so this runs first`); priorities with no `Locking_Policy`/`Queuing_Policy` configuration; separate high/medium entries used to emulate priority queues.

**Apply.** `advisory`.

**Source.** Q&S95 §6.1.6; ARM Annex D.

---

## CON-09 `delay until` for periodic activity; never busy-wait

**Rule.** A `delay` guarantees only a minimum; the task may run much later. For a periodic task compute the next absolute time and `delay until` it, so the period does not drift by the loop's execution time; use `Ada.Real_Time` when precision matters and keep the `Next := Next + Period` arithmetic exact. Never busy-wait, and never wait by polling in a loop with a short `delay` when the expected wait is long: block on an entry, `select` or an appropriate delay instead.

**Detect.** `auto`. `loop ... delay <constant>; ... end loop` with no `delay until` (drifting period); `while not Flag loop null; end loop` or `loop exit when Flag; delay 0.0 .. 0.01; end loop` (busy or near-busy wait); `delay 0.0`.

**Apply.** `auto` for converting a fixed-period `delay` loop to the `delay until` pattern (exact transformation); `assisted` for busy waits, which need a blocking design.

**Example.**
```ada
loop
   delay until Poll_Time;
   ...  --  poll the device
   Poll_Time := Poll_Time + Period;
end loop;
```

**Source.** Q&S95 §6.1.7, §6.2.5; Ada 95 Rationale §9.3; ARM §D.8.

---

## CON-10 Concurrency and type extension: synchronise at the leaves, or generically

**Rule.** Adding a protected-type component to a tagged type freezes its concurrent behaviour for all descendants, so add protected components at the leaves of the hierarchy, or supply synchronisation through a generic wrapper that can be instantiated per application type. Three patterns: a limited tagged root with a protected component and primitives that call its operations; a protected type wrapping a finished tagged type; or a protected type with a tagged component used to implement a new root. (Ada 2005 adds synchronised interfaces, see [14-modern-ada.md](14-modern-ada.md).)

**Detect.** `assisted`. Protected components in a root tagged type with several extensions.

**Apply.** `advisory`.

**Source.** Q&S95 §6.1.8.

---

## CON-11 Keep rendezvous short; keep the select loop short

**Rule.** Inside `accept ... do ... end`, do only what must happen while both tasks are blocked (copy the parameters). Work placed after the accept in the select loop still delays the next caller; if that delay is unacceptable, give the work to another task. Protected operations are cheaper than rendezvous for data communication.

**Detect.** `auto`. `accept` bodies longer than about 10 lines or containing loops, I/O, or calls to other tasks or protected entries.

**Apply.** `assisted`. Move statements out of the accept body after copying the parameters to locals.

**Source.** Q&S95 §6.2.1.

---

## CON-12 Defensive task communication

**Rule.** Wrap every entry call and every rendezvous in handling for `Tasking_Error` (the callee may be abnormal, which the caller cannot prevent). Handle `Program_Error` wherever a selective accept can have all alternatives closed and has no `else` part (an `else` cannot be guarded, but it is mutually exclusive with `delay` and `terminate` alternatives, so it is not always available). Be prepared for an exception raised during a rendezvous: it propagates to both tasks and must be handled in both. A `when others` in the accept keeps unexpected exceptions from reaching callers, but re-raise after handling so the task's outer frame can decide.

**Detect.** `auto`. Entry calls (`\w+\.\w+\s*\(` on a task object, or `\w+\.\w+;`) outside any frame with a `Tasking_Error` handler; `select` with every alternative guarded and no `else`, no `Program_Error` handler.

**Apply.** `assisted`. Add a named block with handlers around the call.

**Source.** Q&S95 §6.2.2, §6.3.1.

---

## CON-13 Do not trust 'Count, 'Callable or 'Terminated on tasks

**Rule.** These attributes are subject to race conditions between evaluation and action. `'Callable` is reliable only once False, `'Terminated` only once True. For task entries never use `'Count` in a guard or to decide anything: a caller can leave the queue by abort or timed-call expiry. `'Count` on a protected entry is stable within a protected action, but a barrier using it is evaluated both before and after queuing a caller. Never test `'Callable`/`'Terminated` to avoid `Tasking_Error`; handle the exception ([CON-12](#con-12-defensive-task-communication)).

**Detect.** `auto`. `'Count` inside a `when` guard of a task `select`; `if \w+'Callable then` / `'Terminated` before an entry call.

**Apply.** `assisted`.

**Source.** Q&S95 §6.2.3, §6.3.4; ARM §9.9.

---

## CON-14 No unprotected shared variables

**Rule.** Pass data between tasks through protected subprograms and entries (or a rendezvous), never through unprotected shared variables, and never use a shared variable as a synchronisation flag. Guards must not reference non-local variables ([CON-02](#con-02-barriers-no-globals-no-side-effects-no-blocking-inside-protected-operations)). Where a shared object is unavoidable (interface to hardware, to non-Ada processes, or deliberately in shared memory), mark it `Volatile` or, if loads and stores must be indivisible, `Atomic` (checking size and alignment). The predefined random number generators and `Text_IO` files are shared state too and need synchronised access.

**Detect.** `auto`. Package-level variables written in one task body and read in another with no `pragma Volatile`/`Atomic` and no protected wrapper; `Ada.Numerics.*_Random` generators used from several tasks.

**Apply.** `auto` for adding `pragma Volatile`/`Atomic` where the project confirms the object is shared; `assisted` for converting to a protected object.

**Source.** Q&S95 §6.2.4; ARM §9.10, §A.5.2, §C.6; Ada 95 Rationale §C.5.

---

## CON-15 Conditional and timed calls and accepts with care

**Rule.** Treat conditional entry calls (`select ... else`), selective accepts with `else` parts, timed entry calls and `delay` alternatives as implementation-dependent and prone to race conditions: in a loop they become busy waits, and "immediately" is undefined. If one is needed in a loop, pair it with a `delay until` to yield the processor. Do not design around a timed call's delay being exact; it is a minimum.

**Detect.** `auto`. `select ... else` inside a `loop` with no `delay`; `or delay <literal>;` used as a watchdog with safety consequences.

**Apply.** `assisted`.

**Source.** Q&S95 §6.2.5.

---

## CON-16 Minimise accepts and selects per task and per entry

**Rule.** Keep the number of `accept` and `select` statements in a task small, and accept each entry in one place, branching on mode inside the accept rather than accepting the same entry in several branches. Only entries that affect the task's externally observable behaviour (timing, ordering, priority, data updates) belong in its interface. Trade this off against guard complexity: separate selects per mode can be clearer (Sanden 1994).

**Detect.** `auto`. Same entry accepted in more than one `accept` statement; tasks with more than about five `select` statements.

**Apply.** `assisted`.

**Source.** Q&S95 §6.2.6.

---

## CON-17 Terminate deliberately

**Rule.** Do not create non-terminating tasks by accident. Every selective accept that has neither an `else` nor a `delay` alternative gets a `terminate` alternative; prefer `select ... or terminate` to a bare `accept`. Tasks that depend on library packages cannot terminate through `terminate` and must be shut down explicitly (an exit entry called by the main subprogram before it ends). Do not create tasks in `Finalize` after the environment task has begun waiting. A deliberately non-terminating task (cyclic executive) must not depend on any block or subprogram whose callers expect to return, and must be documented. Beware that `Ada.Command_Line.Set_Exit_Status` from several tasks races; serialise it.

**Detect.** `auto`. Task bodies with `loop ... select ... end select; end loop;` and no `terminate`, `else` or `delay` alternative; library-level tasks with no shutdown entry; `Set_Exit_Status` called from more than one task.

**Apply.** `assisted`. Adding `or terminate;` to a select is safe when the task has no clean-up to do; otherwise add a shutdown entry.

**Source.** Q&S95 §6.3.2, §6.3.6; ARM §9.3, §10.2.

---

## CON-18 Avoid abort; prefer asynchronous transfer of control, and use even that sparingly

**Rule.** Do not use `abort`: the target's data may be left inconsistent, dependants must detect the termination, and the abort takes effect only at an abort completion point, so it may neither free resources promptly nor stop a runaway loop. Where a computation must be cancellable, use `select ... then abort` (asynchronous select), which keeps one thread and makes the abortable region explicit, and even then keep its use rare. In task bodies and in abortable parts, do not assign to non-atomic global objects (a disrupted assignment makes the object abnormal); assign to locals and use protected objects. Do not use an asynchronous select inside an abort-deferred operation (protected actions, `Initialize`, `Adjust`, `Finalize`), and do not create a task whose master lies wholly within one.

**Detect.** `auto`. `\babort\b` statements; `then abort` blocks that assign package-level variables; `select ... then abort` inside a protected body or a controlled operation.

**Apply.** `assisted`. The guide's pattern: a `Limited_Controlled` transaction object whose `Finalize` commits or rolls back according to an `Atomic` status set by the abortable part.

**Source.** Q&S95 §6.3.3, §5.9.10; Ada 95 Rationale §9.3, §9.4; ARM §9.8.

---

## CON-19 `when others` at the end of every task body

**Rule.** End every task body with an exception handler, including `when others`, that reports the task's demise (to a log, a status table, or a supervisor) and shuts the task down cleanly. An unhandled exception in a task silently kills it; nothing else in the program is told. Put a handler for the rendezvous inside the main loop too, so a failed entry call (for example a degraded sensor) can switch mode rather than end the task ([CON-12](#con-12-defensive-task-communication)).

**Detect.** `auto`. `task body` whose outermost frame has no `exception` section, or whose `when others` is `null`.

**Apply.** `auto` for adding the handler skeleton with the project's logging call; `assisted` for what recovery to perform.

**Example.**
```ada
task body Track is
begin
   ...
exception
   when E : others =>
      Station (My_Index).Status := Dead;
      Log ("Track " & My_Index'Image & " died: " & Exception_Information (E));
end Track;
```

**Source.** Q&S95 §6.3.4, §6.3.1.

---

## CON-20 No circular entry calls

**Rule.** Never call an entry that leads, directly or through a chain, back to an entry of the calling task: it deadlocks.

**Detect.** `assisted`. Build the entry-call graph between tasks and look for cycles.

**Apply.** `advisory`.

**Source.** Q&S95 §6.3.5.

---

## Checklist

| ID | Rule | Apply |
|----|------|-------|
| CON-01 | Protected objects for shared data and passive synchronisation | advisory |
| CON-02 | Barriers local and pure; nothing blocking in protected operations | assisted |
| CON-03 | Protected-procedure interrupt handlers with adequate ceiling | assisted |
| CON-04 | Tasks only for real threads of control; rendezvous for synchronous coupling | advisory |
| CON-05 | Discriminants, not an initial rendezvous | assisted |
| CON-06 | Single declarations for unique instances | assisted |
| CON-07 | Reuse tasks; discipline for dynamic ones | advisory |
| CON-08 | Priorities are not synchronisation | advisory |
| CON-09 | `delay until` for periodic loops; no busy wait | auto/assisted |
| CON-10 | Synchronise at the leaves of a hierarchy or via generics | advisory |
| CON-11 | Short rendezvous, short select loop | assisted |
| CON-12 | Handle `Tasking_Error` and `Program_Error` around communication | assisted |
| CON-13 | Do not trust `'Count`, `'Callable`, `'Terminated` on tasks | assisted |
| CON-14 | No unprotected shared variables; `Volatile`/`Atomic` when unavoidable | auto/assisted |
| CON-15 | Conditional and timed calls with care; never in a tight loop | assisted |
| CON-16 | Few accepts and selects; one accept per entry | assisted |
| CON-17 | Deliberate termination: `terminate` alternatives, explicit shutdown of library tasks | assisted |
| CON-18 | No `abort`; rare asynchronous select; no global assignment in abortable parts | assisted |
| CON-19 | `when others` at the end of every task body, reporting the demise | auto |
| CON-20 | No circular entry calls | advisory |
