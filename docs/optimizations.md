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

---

## Principles

All optimizations must satisfy:

1. **Semantics preservation.** Optimized code must produce the same observable behavior as unoptimized code for
   every valid NL program. Output, exceptions, return values, and side effects must be indistinguishable.

2. **Side-effect ordering.** The relative order of side effects (I/O, exceptions, mutations visible to other
   threads) must not change. Reordering of independent pure computations is allowed; reordering across side effects
   is not.

3. **Implementation freedom.** Unless otherwise specified, optimizations are **optional** (`may`). Implementations
   are free to apply them or not. Correctness must never depend on a specific optimization being applied.

4. **Portability.** Programs must behave correctly regardless of which optimizations an implementation applies.
   The only observable differences may be performance and resource usage (e.g., memory, CPU).

---

## Compiler optimizations

The following optimizations are **optional** (`may`). The compiler may apply them when beneficial.

| Optimization | Description |
|--------------|-------------|
| **Constant folding** | Evaluate constant expressions at compile time: `2 + 3` → `5`, `"a" + "b"` → `"ab"`. Emit the result as a constant instead of runtime computation. |
| **Constant propagation** | Propagate known constant values through variables to enable further constant folding or dead code elimination. |
| **Dead code elimination** | Remove code that is never executed: unreachable code after `return`, `throw`, or in branches that are statically known to be false. |
| **Devirtualization** | Replace virtual dispatch with direct calls when the receiver's static type is known and the method is `final`, or when the class is not extended. See [vm.md § Instance methods](vm.md#instance-methods). |
| **Inlining** | Replace a call site with the callee's body for small methods, `static` methods, or when heuristics suggest benefit. |
| **Tail call optimization** | Reuse the current call frame for recursive calls in tail position, avoiding stack growth. |
| **String literal concatenation** | Fold `"a" + "b"` when both operands are string literals into a single constant pool entry. |

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

---

## Observability

**Observable behavior** includes:

- **Return values** of `main` and of any function whose result is used.
- **Output** to stdout, stderr, or files (via `system.Out`, `system.io.File`, etc.).
- **Exceptions** thrown and their propagation (including stack traces).
- **Destructor invocations** (see [vm.md § Garbage collection contract](vm.md#garbage-collection-contract)):
  destructors must run before object reclamation; their order for unreachable objects is implementation-defined.

**Not observable** (and thus may be optimized freely):

- Allocation patterns (except as required for correctness).
- Number of bytecode instructions executed.
- Internal representation of values (e.g., whether strings are interned).
- Timing and CPU usage (unless specified elsewhere).

---

## Testing

- **Regression tests:** Run the same program with and without optimizations; compare outputs, exit codes, and
  exception behavior. All existing tests in `tests/` must pass regardless of optimization level.
- **Performance tests:** Optional; may be defined in a separate document or in [milestones.md](milestones.md).
