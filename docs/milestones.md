# NL — Implementation Milestones

This document describes the recommended phases for building a complete NL toolchain (compiler + VM + test runner). Each milestone produces a testable deliverable and builds on the previous ones. References point to the specific sections of the specification documents.

> **Note:** This is a suggested roadmap, not a rigid plan. Phases can overlap and the order may be adjusted as the implementation progresses.

---

## Summary

| # | Milestone | Deliverable | Key references |
|---|-----------|-------------|----------------|
| 1 | [Lexer & Parser](#milestone-1--lexer--parser) | AST for all NL syntax | specs.md |
| 2 | [Semantic analysis](#milestone-2--semantic-analysis) | Error diagnostics (39 error codes, 1 warning) | compiler.md, specs.md |
| 3 | [Bytecode emission](#milestone-3--bytecode-emission) | Valid `.nlm` modules | vm.md §§ Module format, Compilation strategies; compiler.md § [Compiler invocation (nlc)](compiler.md#compiler-invocation-nlc) |
| 4 | [VM core](#milestone-4--vm-core) | Execute primitive programs | vm.md §§ Architecture, Value representation, Instruction set, Program startup, [VM invocation (nlvm)](vm.md#vm-invocation-nlvm) |
| 5 | [Objects, arrays & dispatch](#milestone-5--objects-arrays--dispatch) | OOP programs run | vm.md §§ Object model, Method dispatch |
| 6 | [Exceptions & closures](#milestone-6--exceptions--closures) | Full control flow | vm.md §§ Exception handling, Closures |
| 7 | [Standard library](#milestone-7--standard-library) | Native bindings for `system.*` | stdlib.md, vm.md § Standard library binding |
| 8 | [Test runner & integration](#milestone-8--test-runner--integration) | All `tests/` YAML files pass | tests.md |
| 9 | [Optimizations](#milestone-9--optimizations) | Optional compiler and VM optimizations | optimizations.md |

---

## Milestone 1 — Lexer & Parser

Build the front-end that transforms NL source text into an abstract syntax tree (AST).

### Scope

- **Lexer:** tokenize all keywords ([specs.md § Keywords](specs.md#keywords)), identifiers, literals (int, float, string, bool), operators, and punctuation.
- **Parser:** produce an AST covering:
  - Namespace declarations and imports (`use`, `as`) — [specs.md § Source code files](specs.md#source-code-files), [§ Imports](specs.md#imports).
  - Class declarations (fields, methods, visibility, `readonly`, `const`, `static`, `nodiscard`) — [specs.md § Classes](specs.md#classes).
  - Constructors and destructors — [specs.md § Constructors and Destructors](specs.md#constructors-and-destructors).
  - Template classes and methods — [specs.md § Template class](specs.md#template-class), [§ Template methods](specs.md#template-methods).
  - Inheritance and interfaces (`extends`, `implements`) — [specs.md § Extends, Implements](specs.md#extends-implements).
  - Enums (basic, typed, with methods) — [specs.md § Enums](specs.md#enums).
  - All expressions: arithmetic, comparison, logical, string concatenation, `??`, `?:`, casts, `instanceof` — [specs.md § Operators](specs.md#operators).
  - Operator overloading declarations — [specs.md § Operator Overloading](specs.md#operator-overloading).
  - Control structures: `if`/`else`, `while`, `for` (classic and for-each), `switch`/`match` — [specs.md § Control Structures](specs.md#control-structures).
  - Anonymous functions (closures) — [specs.md § Anonymous Functions](specs.md#anonymous-functions).
  - Exception syntax: `try`/`catch`/`finally`, `throw`, `throws` — [specs.md § Exceptions](specs.md#exceptions).
  - Types: primitives, arrays `T[]`, unions `T|null`, `auto`, `typedef` — [specs.md § Types](specs.md#types).
  - Parameter modifiers: `ref`, `const`, `const ref`, named parameters, optional parameters — [specs.md § Parameter passing semantics](specs.md#parameter-passing-semantics), [§ Named parameters](specs.md#named-parameters).
  - Entry point signature — [specs.md § Entry point](specs.md#entry-point).
- **Multi-file loading:** read source blocks from YAML test files or from disk following the one-class-per-file convention.

### Testable at this stage

- Parse every `.nl` source block in `tests/` without error.
- Round-trip or AST-dump tests for each syntactic construct.

---

## Milestone 2 — Semantic analysis

Implement all compile-time checks defined in compiler.md. This milestone requires name resolution and a type system over the AST.

### Scope

- **Name resolution:** resolve namespace-qualified names, imports, class references, field/method lookups, and `super`/`this`/`Self`/`type` keywords.
- **Type system:** primitives, arrays, union types (`T|null`), `auto` deduction, `typedef` aliases, class/interface subtyping, template instantiation (monomorphization).
- **Definite assignment analysis** — [compiler.md § Definite assignment](compiler.md#definite-assignment-analysis): E001, E002.
- **Null safety** — [compiler.md § Null safety](compiler.md#null-safety): E003, E004.
- **Type checking** — [compiler.md § Type checking](compiler.md#type-checking): `auto` (E005), templates (E006, E037), casts (E007), string concatenation (E008), operator compatibility (E009).
- **Immutability enforcement** — [compiler.md § Immutability](compiler.md#immutability-enforcement): `const` methods (E010, E011), `const` parameters (E012), for-each in const context (E039), `readonly` (E013, E014).
- **Exception checking** — [compiler.md § Exception checking](compiler.md#exception-checking): checked propagation (E015), inheritance rules (E016, E017).
- **Visibility enforcement** — [compiler.md § Visibility](compiler.md#visibility-enforcement): E018, E019.
- **Parameter validation** — [compiler.md § Parameter validation](compiler.md#parameter-validation): `ref` rules (E020–E022), named/optional rules (E023–E026).
- **Entry point validation** — [compiler.md § Entry point](compiler.md#entry-point-validation): E027–E029.
- **Inheritance modifiers** — [compiler.md § Inheritance modifiers](compiler.md#inheritance-modifiers-abstract-final): abstract (E032–E034), final (E035, E036).
- **Reserved keywords** — [compiler.md § Reserved keywords](compiler.md#reserved-keywords): E030.
- **Default values and arrays** — [compiler.md § Default values](compiler.md#default-values): E031, E038.
- **Warnings:** `nodiscard` — [compiler.md § Nodiscard](compiler.md#nodiscard): W001.

### Testable at this stage

- **compile-only** tests (`compile_only: true` in YAML): the compiler accepts valid programs and rejects invalid ones with the correct error codes.
- Unit tests per error code: one positive case (accepted) and one negative case (rejected with expected code).

---

## Milestone 3 — Bytecode emission

Generate the binary module format defined in vm.md from the validated AST.

### Scope

- **Module format** — [vm.md § Module format](vm.md#module-format): magic number, version, constant pool, class/field/method descriptors, flags.
- **Constant pool** — [vm.md § Constant pool](vm.md#constant-pool): INT, FLOAT, STRING, CLASS, FIELD_REF, METHOD_REF, TYPE_DESC entries.
- **Type descriptors** — [vm.md § Type descriptors](vm.md#type-descriptors): encoding for all types.
- **Instruction emission** for all compilation strategies — [vm.md § Compilation strategies](vm.md#compilation-strategies):
  - Templates (monomorphization) — [vm.md § Templates](vm.md#templates-monomorphization).
  - `ref` parameters (boxing) — [vm.md § Ref parameters](vm.md#ref-parameters-boxing).
  - String concatenation (`TO_STRING` + `STR_CONCAT`) — [vm.md § String concatenation](vm.md#string-concatenation).
  - Union types — [vm.md § Union types at runtime](vm.md#union-types-at-runtime).
  - For-each loops (desugaring) — [vm.md § For-each loops](vm.md#for-each-loops).
  - Named and optional parameters (reordering + defaults) — [vm.md § Named and optional parameters](vm.md#named-and-optional-parameters).
  - Operator overloading → `INVOKE_INSTANCE`/`INVOKE_STATIC` — [vm.md § Operator overloading](vm.md#operator-overloading).
  - `++`/`--` strategies — [vm.md § Increment and decrement operators](vm.md#increment-and-decrement-operators).
  - `??` and `?:` — [vm.md § Nullish coalescing and elvis operators](vm.md#nullish-coalescing-and-elvis-operators).
  - `switch`/`match` — [vm.md § Switch statements](vm.md#switch-statements), [§ Match expressions](vm.md#match-expressions).
- **Method descriptors:** `max_locals`, `max_stack` computation, exception table generation.

### Testable at this stage

- Module-structure tests from YAML: `expected_class`, `expected_methods`, `expected_fields`, `expected_constant_pool_contains` — [tests.md § Header keys](tests.md#header-keys).
- Binary round-trip: emit a module, re-read it, verify structure.

---

## Milestone 4 — VM core

Build the execution engine for primitive programs (no objects, no exceptions yet).

### Scope

- **Module loader:** read the binary module format, parse constant pool, build internal class table.
- **Value representation** — [vm.md § Value representation](vm.md#value-representation): tagged values (9-byte slots), primitive type tags.
- **Call frame and operand stack** — [vm.md § Call frame and operand stack](vm.md#call-frame-and-operand-stack): frame creation, local variables, stack pointer.
- **Static storage** — [vm.md § Static storage](vm.md#static-storage): per-class static fields, default initialization.
- **Instruction set (subset):**
  - Stack manipulation: `NOP`, `POP`, `DUP`, `SWAP`, `DUP_X1`.
  - Constants: `CONST_NULL`, `CONST_TRUE`, `CONST_FALSE`, `CONST_IZERO`, `CONST_IONE`, `CONST_FZERO`, `CONST_FONE`, `BIPUSH`, `SIPUSH`, `LDC`.
  - Local variable access: `LOAD`, `STORE`, `LOAD_0`–`LOAD_3`, `STORE_0`–`STORE_3`.
  - Integer arithmetic: `IADD`, `ISUB`, `IMUL`, `IDIV`, `IMOD`, `INEG`, `IINC`.
  - Float arithmetic: `FADD`, `FSUB`, `FMUL`, `FDIV`, `FMOD`, `FNEG`.
  - Type conversions: `I2F`, `F2I`, `I2B`, `B2I`, `TO_STRING`.
  - Comparison: `CMP_EQ`, `CMP_NE`, `CMP_LT`, `CMP_GT`, `CMP_LE`, `CMP_GE`, `CMP_THREE_WAY`, `IS_NULL`, `IS_NONNULL`, `NOT`.
  - Control flow: `IF_TRUE`, `IF_FALSE`, `GOTO`, `GOTO_W`.
  - Method invocation: `INVOKE_STATIC`, `RETURN`, `RETURN_VALUE`.
  - String operations: `STR_CONCAT`.
- **Program startup (partial)** — [vm.md § Program startup](vm.md#program-startup): load entry module, build `args`, invoke `main`, return exit code.

### Testable at this stage

- **Run** tests (`expected_exit_code`, `expected_stdout`) for programs using only static methods, primitives, strings, and control flow.
- Arithmetic and comparison correctness tests.

---

## Milestone 5 — Objects, arrays & dispatch

Add the object model and method dispatch.

### Scope

- **Object operations:**
  - `NEW` — heap allocation, default field initialization.
  - `INSTANCEOF`, `CHECKCAST` — runtime type checks.
- **Field access:** `GET_FIELD`, `SET_FIELD`, `GET_STATIC`, `SET_STATIC`, readonly enforcement at runtime — [vm.md § Field access](vm.md#field-access).
- **Object layout** — [vm.md § Object layout](vm.md#object-layout): header, inherited fields, declared fields.
- **Array operations:** `NEW_ARRAY`, `NEW_ARRAY_INIT`, `ARRAY_LOAD`, `ARRAY_STORE`, `ARRAY_LENGTH` — [vm.md § Array layout](vm.md#array-layout).
- **String representation** — [vm.md § String representation](vm.md#string-representation): immutable heap objects, interning.
- **Enum representation** — [vm.md § Enum representation](vm.md#enum-representation): static readonly fields, `from()`/`tryFrom()` as static methods.
- **Method dispatch** — [vm.md § Method dispatch](vm.md#method-dispatch):
  - `INVOKE_INSTANCE` — virtual dispatch via vtable.
  - `INVOKE_SPECIAL` — constructors, `super` calls, private methods.
  - Interface dispatch.
- **Constructors and destructors** — [vm.md § Constructors](vm.md#constructors): `NEW` + `DUP` + `INVOKE_SPECIAL` pattern.

### Testable at this stage

- Run tests with class instantiation, field access, inheritance, method overriding.
- Array creation and element access.
- Enum `from()` / `tryFrom()`.
- Virtual dispatch tests (calling overridden methods through parent reference).

---

## Milestone 6 — Exceptions & closures

Complete the control flow model.

### Scope

- **Exception handling** — [vm.md § Exception handling](vm.md#exception-handling):
  - Exception table lookup (`start_pc`, `end_pc`, `handler_pc`, `catch_type`).
  - `THROW` opcode and stack unwinding.
  - `finally` blocks (catch-all entries, code duplication).
  - Stack trace construction — [vm.md § Stack trace construction](vm.md#stack-trace-construction): `ExecutionPoint` with `line` and `file`.
  - Implicit exceptions: `ArithmeticException` (division by zero), `NullPointerException`, `IndexOutOfBoundsException`, `InvalidCastException`.
- **Closures** — [vm.md § Closures and anonymous functions](vm.md#closures-and-anonymous-functions):
  - Synthetic closure classes (captured fields + invoke method).
  - Variable capture and boxing for mutable captures.
  - `INVOKE_CLOSURE` opcode.
  - Type compatibility with `typedef`'d function types.

### Testable at this stage

- `try`/`catch`/`finally` tests: caught exceptions, uncaught propagation, nested try blocks.
- Stack trace content verification (via `expected_stderr` or program-level output).
- Anonymous function tests: capture, mutation, passing as callbacks.

---

## Milestone 7 — Standard library

Implement native bindings for all `system.*` classes.

### Scope

- **Native module binding** — [vm.md § Standard library binding](vm.md#standard-library-binding): the VM intercepts `INVOKE_STATIC`/`INVOKE_INSTANCE` on `system.*` classes.
- **String instance methods** — [stdlib.md § system.String](stdlib.md#systemstring): `length`, `charAt`, `substring`, `indexOf`, `contains`, `replace`, `split`, `trim`, `toUpperCase`, `toLowerCase`, `startsWith`, `endsWith`.
- **Array built-in methods** — [specs.md § Arrays](specs.md#arrays) / [vm.md § Standard library binding](vm.md#standard-library-binding): `slice`, `map`, `filter`, `forEach`, `sort`, `find` (native, use `INVOKE_CLOSURE` for callbacks).
- **I/O:** `system.Out`, `system.Err`, `system.In` — [stdlib.md § system.Out](stdlib.md#systemout-stdout), [§ system.Err](stdlib.md#systemerr-stderr), [§ system.In](stdlib.md#systemin-stdin).
- **Parsing:** `system.Int`, `system.Float`, `system.Bool` — [stdlib.md § system.Int](stdlib.md#systemint), [§ system.Float](stdlib.md#systemfloat), [§ system.Bool](stdlib.md#systembool).
- **Collections:** `system.List<T>`, `system.Map<K,V>` — [stdlib.md § system.List](stdlib.md#systemlist), [§ system.Map](stdlib.md#systemmap). Template instantiation via naming convention (`"system.List<int>"`).
- **File system:** `system.io.File`, `system.io.FileHandle`, `system.io.Directory`, `system.io.Path`, `system.io.Grep` — [stdlib.md § system.io](stdlib.md#systemiofile).
- **Network:** `system.net.TcpListener`, `system.net.TcpStream`, `system.net.UdpSocket`, `system.net.Http` — [stdlib.md § system.net](stdlib.md#systemnettcplistener).
- **Threading:** `system.thread.Thread`, `system.thread.Mutex`, `system.thread.Semaphore` — [stdlib.md § system.thread](stdlib.md#systemthreadthread). Memory visibility rules — [vm.md § Threading model](vm.md#threading-model).
- **Date/Time:** `system.time.DateTime`, `system.time.TimeZone` — [stdlib.md § system.time](stdlib.md#systemtimedatetime).
- **Process:** `system.ps.Process` — [stdlib.md § system.ps](stdlib.md#systemps).
- **Text:** `system.text.Regex`, `system.text.Encoding` — [stdlib.md § system.text](stdlib.md#systemtextregex).
- **Other:** `system.Random`, `system.Uuid`, `system.Env` — [stdlib.md § system.Random](stdlib.md#systemrandom), [§ system.Uuid](stdlib.md#systemuuid), [§ system.Env](stdlib.md#systemenv).
- **Garbage collection** — [vm.md § Garbage collection contract](vm.md#garbage-collection-contract): destructor calls, reachability.

### Testable at this stage

- Programs using `system.Out.println`, file I/O, string parsing, collections.
- Multi-threaded programs with mutexes.
- Subprocess execution via `system.ps.Process.run`.

---

## Milestone 8 — Test runner & integration

Build the test runner that executes the YAML test suite, and validate the full toolchain.

### Scope

- **YAML parser** — [tests.md](tests.md): read front matter (all header keys), extract source blocks by `file_separator`.
- **Run tests:** compile all source blocks, execute the program, compare `expected_exit_code` and `expected_stdout` (and `expected_stderr` if present).
- **Compile-only tests** (`compile_only: true`): compile and verify success, no execution.
- **Module-structure assertions:** parse the compiled module and verify `expected_class`, `expected_methods`, `expected_fields`, `expected_constant_pool_contains`.
- **Test discovery:** scan `tests/` directory, run all `*.yaml` files, report pass/fail summary.
- **Error tests (extension):** tests that verify the compiler correctly rejects invalid programs with specific error codes (E001–E039, W001). Requires extending the test format or adding a convention (e.g. `expected_error: "E003"` header key).

### Testable at this stage

- All `tests/*.yaml` files pass.
- CI-ready: the test runner returns exit code 0 when all tests pass, non-zero otherwise.

---

## Milestone 9 — Optimizations

Implement optional compiler and VM optimizations as defined in [optimizations.md](optimizations.md). This milestone is
**optional** and can be pursued after the core toolchain is stable.

### Scope

- **Compiler optimizations** — [optimizations.md § Compiler optimizations](optimizations.md#compiler-optimizations):
  constant folding, constant propagation, dead code elimination, devirtualization, inlining, tail call optimization,
  string literal concatenation.
- **VM optimizations** — [optimizations.md § VM optimizations](optimizations.md#vm-optimizations): string interning,
  JIT compilation, superinstructions, inline caching, GC tuning.
- **Prohibited transformations** — [optimizations.md § Prohibited transformations](optimizations.md#prohibited-transformations):
  ensure no reordering of side effects, no elimination of observable calls, no fusion that changes semantics.

### Testable at this stage

- **Regression tests:** run all `tests/` programs with and without optimizations; compare outputs, exit codes, and
  exception behavior. All tests must pass regardless of optimization level.
- **Performance benchmarks:** optional; compare execution time or memory usage with/without optimizations.

---

## Dependency graph

```
1. Lexer & Parser
       │
       ▼
2. Semantic Analysis
       │
       ▼
3. Bytecode Emission ──────────────────┐
       │                               │
       ▼                               ▼
4. VM Core                   8. Test Runner (module-structure
       │                        assertions available here)
       ▼
5. Objects, Arrays & Dispatch
       │
       ▼
6. Exceptions & Closures
       │
       ▼
7. Standard Library
       │
       ▼
8. Test Runner (run tests available here)
       │
       ▼
9. Optimizations (optional)
```

Milestones 1–3 are the **compiler** track. Milestones 4–7 are the **VM** track. Milestone 8 ties everything together. Milestone 9 (optimizations) is optional and builds on the complete toolchain. Module-structure tests (milestone 3 deliverables) can be validated by a partial test runner as early as milestone 3; full run tests require milestone 7.
