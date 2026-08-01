# NL — Optimization Contract

This document describes the **optimization contract** for the NL compiler and virtual machine. It defines which
optimizations implementations **may** apply, which transformations are **prohibited**, and the principles that
govern all optimizations. It complements [specs.md](specs.md) (language semantics), [compiler.md](compiler.md)
(compile-time checks), and [vm.md](vm.md) (execution model).

## Summary

* [Principles](#principles)
* [Compiler optimizations](#compiler-optimizations)
* [VM optimizations](#vm-optimizations)
* [Prohibited transformations](#prohibited-transformations)
* [Observability](#observability)
    * [Stack traces and frame-eliding optimizations](#stack-traces-and-frame-eliding-optimizations)
    * [Tail call optimization and `StackOverflowException`](#tail-call-optimization-and-stackoverflowexception)
* [Testing](#testing)

---

## Principles

All optimizations must satisfy:

1. **Semantics preservation.** Optimized code must produce the same observable behavior as unoptimized code for
   every valid NL program. Output, exceptions, return values, and side effects must be indistinguishable.
   *Observable behavior* is defined by [§ Observability](#observability); what that section lists as **not
   observable** — including the frames of a stack trace and the depth of the call stack — is outside this
   guarantee.

2. **Side-effect ordering.** The relative order of side effects (I/O, exceptions, mutations visible to other
   threads) must not change. Reordering of independent pure computations is allowed; reordering across side effects
   is not.

3. **Implementation freedom.** Unless otherwise specified, optimizations are **optional** (`may`). Implementations
   are free to apply them or not. Correctness must never depend on a specific optimization being applied.

4. **Portability.** Programs must behave correctly regardless of which optimizations an implementation applies.
   The only observable differences may be performance and resource usage (e.g., memory, CPU). A program whose
   result depends on a value that [§ Observability](#observability) declares implementation-defined — such as
   `stackTrace.length()` — is not portable, and the specification promises nothing about it.

---

## Compiler optimizations

The following optimizations are **optional** (`may`). The compiler may apply them when beneficial.

| Optimization | Description |
|--------------|-------------|
| **Constant folding** | Evaluate constant expressions at compile time: `2 + 3` → `5`, `"a" + "b"` → `"ab"`. Emit the result as a constant instead of runtime computation. |
| **Constant propagation** | Propagate known constant values through variables to enable further constant folding or dead code elimination. |
| **Dead code elimination** | Remove code that is never executed: unreachable code after `return`, `throw`, or in branches that are statically known to be false. |
| **Devirtualization** | Replace virtual dispatch with direct calls when the receiver's static type is known and the method is `final`, or when the class is not extended. See [vm.md § Instance methods](vm.md#instance-methods). |
| **Inlining** | Replace a call site with the callee's body for small methods, `static` methods, or when heuristics suggest benefit. Removes the callee's call frame; see [Stack traces and frame-eliding optimizations](#stack-traces-and-frame-eliding-optimizations). |
| **Tail call optimization** | Reuse the current call frame for recursive calls in tail position, avoiding stack growth. Removes call frames and may prevent `StackOverflowException`; see [Stack traces and frame-eliding optimizations](#stack-traces-and-frame-eliding-optimizations) and [Tail call optimization and `StackOverflowException`](#tail-call-optimization-and-stackoverflowexception). |
| **String literal concatenation** | Fold `"a" + "b"` when both operands are string literals into a single constant pool entry. |
| **Incremental compilation** | Cache compiled modules per source file; recompile only modified files and their dependents (transitively). Uses the module-per-file model (see [vm.md § Module format](vm.md#module-format)) and explicit `use` dependencies (see [specs.md § Imports](specs.md#imports)). Cache invalidation (hash, mtime), cache location, and template instantiation handling are implementation-defined. |

---

## VM optimizations

The following optimizations are **optional** (`may`). The VM may apply them at load time or during execution.

| Optimization | Description |
|--------------|-------------|
| **String interning** | Share identical string literals from the constant pool so they refer to the same heap object. Correctness must not depend on it; string equality is always by content. See [vm.md § String representation](vm.md#string-representation). |
| **JIT compilation** | Compile frequently executed bytecode to native code for faster execution. |
| **Superinstructions** | Fuse common opcode sequences (e.g., `LOAD` + `IADD`) into single interpreted steps to reduce dispatch overhead. |
| **Inline caching** | Cache the result of method dispatch for polymorphic call sites to speed up subsequent invocations. |
| **GC tuning** | Use generational, incremental, or other GC strategies. The GC algorithm is implementation-defined; see [vm.md § Garbage collection contract](vm.md#garbage-collection-contract). |

---

## Prohibited transformations

The following transformations are **not allowed**:

1. **Reordering of side effects.** Do not reorder I/O, exception throws, or mutations that are visible to other
   threads. The observable order of side effects must match program order.

2. **Elimination of observable calls.** Do not remove or hoist calls whose only effect is to throw an exception
   or perform I/O, even if the result is unused.

3. **Fusion that changes semantics.** Do not fuse loops or blocks in a way that alters the order or number of
   side effects (e.g., merging two loops that each perform I/O into one that runs in a different order).

4. **Breaking volatile/atomic semantics.** When volatile or atomic access is specified in the future, optimizations
   must preserve the defined memory visibility and ordering guarantees.

5. **Fabricating or reordering call frames.** Frame-eliding optimizations *may* remove frames from a stack trace
   (see [Observability](#observability)), but must never report a frame for a call the source program does not
   make, and must never report frames out of innermost-to-outermost order.

---

## Observability

**Observable behavior** includes:

- **Return values** of `main` and of any function whose result is used.
- **Output** to stdout, stderr, or files (via `system.Out`, `system.io.File`, etc.).
- **Exceptions** thrown and their propagation: which exception types are raised, in which order, at which point in
  the program, and which handler catches them. The *frames* of an exception's stack trace are **not** part of this
  guarantee — see [Stack traces and frame-eliding optimizations](#stack-traces-and-frame-eliding-optimizations).
- **Destructor invocations** (see [vm.md § Garbage collection contract](vm.md#garbage-collection-contract)):
  destructors must run before object reclamation; their order for unreachable objects is implementation-defined.

**Not observable** (and thus may be optimized freely):

- Allocation patterns (except as required for correctness).
- Number of bytecode instructions executed.
- Internal representation of values (e.g., whether strings are interned).
- Timing and CPU usage (unless specified elsewhere).
- **Call stack depth**, and consequently the number and identity of the frames in `Exception.stackTrace` — see
  below.

### Stack traces and frame-eliding optimizations

**Inlining** and **tail call optimization** both remove call frames an unoptimized build would have created.
`Exception.stackTrace` is built by natively walking the *live* call stack during the base `Exception` constructor
(see [vm.md § Stack trace construction](vm.md#stack-trace-construction)), so a frame that an optimization elided
is simply not there to walk. The number and identity of the frames in `stackTrace` are therefore
**implementation-defined**, in the same way their line numbers already are: a stripped build omits the
line-number table and reports `line = 0` (see [vm.md § Method descriptor](vm.md#method-descriptor)).

Reconstructing elided frames from inlining metadata (as HotSpot and V8 do for deoptimization) is a
quality-of-implementation matter, not a conformance requirement. Whatever an implementation chooses, the
following hold at **every** optimization level:

1. **Never empty.** The trace contains at least the frame in which the exception object was constructed.
2. **Innermost first.** Frames appear in the order the corresponding calls sit on the call stack, `new` site
   first — the same order [`printStackTrace()`](specs.md#exception-class-hierarchy) prints.
3. **Never fabricated.** Every `ExecutionPoint` must correspond to a call the source program actually makes —
   either a frame that is really on the stack, or a frame the implementation reconstructed from optimization
   metadata for a call that would be on the stack in an unoptimized build. Frames may be omitted; calls that
   never happened may not be reported.

`stackTrace` is readable by NL programs, not only printed, so this narrowing constrains portable programs: a
program **must not** use `stackTrace.length()`, or the `line`/`file` of a particular frame, to drive control flow
or to compute a result, and must not assume the trace matches what an unoptimized build produces. Diagnostics —
logging, error reporting, `printStackTrace()` — are the intended use.

### Tail call optimization and `StackOverflowException`

Tail call optimization also affects **whether an exception is thrown at all**. Unbounded recursion in tail
position exhausts the call stack, and therefore throws `StackOverflowException` (see
[vm.md § Call frame and operand stack](vm.md#call-frame-and-operand-stack)), in a build that does not apply the
optimization; in a build that does, the same program does not grow the stack and never throws.

This divergence is **permitted**. The VM's maximum call stack depth is explicitly implementation-defined, so no
NL program can rely on a given recursion depth overflowing — TCO makes the difference unbounded rather than
introducing a new kind of divergence. Concretely:

- `StackOverflowException` is a resource-exhaustion signal, not a control-flow mechanism. A program that relies
  on it to end a tail-recursive loop is not portable and may run forever under TCO.
- This is the **only** sanctioned way for the set of exceptions thrown to differ between optimization levels.
  Every other exception must be thrown identically at every optimization level (principle 1).
- An implementation that applies TCO should document it alongside its maximum call stack depth, which
  [vm.md § Call frame and operand stack](vm.md#call-frame-and-operand-stack) already requires it to document.

---

## Testing

- **Regression tests:** Run the same program with and without optimizations; compare outputs, exit codes, and
  exception behavior. All existing tests in `tests/` must pass regardless of optimization level.
- **Optimization-sensitive assertions:** Conformance tests must not assert an exact stack-trace frame count, nor
  expect `StackOverflowException` from recursion in tail position — both are implementation-defined
  (see [Observability](#observability)). A test that verifies a stack trace should assert the presence and
  relative order of the frames it cares about.
- **Performance tests:** Optional; may be defined in a separate document or in [milestones.md](milestones.md).
