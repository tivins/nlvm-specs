# NL — Language Specification

**NL** is a statically-typed, object-oriented programming language designed for type safety and clarity. It draws from C++ (templates, type system), Java (syntax, file layout), modern PHP (match, typed enums), and Kotlin-style null safety with compile-time initialization guarantees.

This repository holds the **complete specification** of NL: language semantics, compiler checks, standard library API, and virtual machine (bytecode and execution model). There is **no compiler or interpreter implementation** here — only the documentation needed to build one.

## Documentation

| Document | Description |
|----------|--------------|
| **[docs/specs.md](docs/specs.md)** | **Language specification.** Lexical rules, types, classes, enums, control flow, operators, operator overloading, anonymous functions, exceptions, and entry point. Defines *what* the language allows. |
| **[docs/compiler.md](docs/compiler.md)** | **Compiler semantic analyses.** Definite assignment, null safety, type checking, immutability (`const` / `readonly`), exception checking, visibility, parameter validation, entry-point validation. Defines *what* the compiler must verify and reject (37 error codes, 1 warning). § [Compiler invocation (nlc)](docs/compiler.md#compiler-invocation-nlc) defines the compiler CLI. |
| **[docs/stdlib.md](docs/stdlib.md)** | **Standard library API.** System namespaces (`system`, `system.io`, `system.net`, `system.thread`, `system.time`, etc.): streams, parsing, files, directories, network (TCP/UDP/HTTP), threads, mutex/semaphore, date/time, `system.Env`, `system.ps.Process`, regex, encoding. Defines the contract between user code and the runtime. |
| **[docs/vm.md](docs/vm.md)** | **Virtual machine specification.** Stack-based execution model, value representation, binary module format (constant pool, class/field/method descriptors), object/array/string layout, instruction set (~50 opcodes), method dispatch (vtables), exception handling (tables, unwinding, stack trace), closures, compilation strategies (templates, ref params, switch/match, etc.), program startup, stdlib binding, threading, and GC contract. Defines *how* compiled code is represented and executed. § [VM invocation (nlvm)](docs/vm.md#vm-invocation-nlvm) defines the VM CLI. |
| **[docs/tests.md](docs/tests.md)** | **Test format.** Structure of test files in `tests/` (YAML front matter, header keys, source blocks with `#NLFILE`-style separators, run vs compile-only). For use by a future compiler/tooling. |
| **[docs/milestones.md](docs/milestones.md)** | **Implementation milestones.** Recommended phases for building the compiler, VM, and test runner — from lexer/parser to full integration. Each milestone lists its scope, spec references, and what becomes testable. |

Read **specs.md** first for the language; then **compiler.md** for compile-time rules, **stdlib.md** for the runtime API, **vm.md** for implementing an interpreter or code generator, and **milestones.md** for a suggested implementation roadmap.

## Project structure

```
nlvm/
├── docs/
│   ├── specs.md      # Language specification
│   ├── compiler.md   # Compiler checks and error codes
│   ├── stdlib.md     # Standard library
│   ├── tests.md      # Test file format (YAML, source blocks)
│   ├── vm.md         # VM and bytecode specification
│   └── milestones.md # Implementation roadmap (8 phases)
├── tests/            # Example / acceptance test definitions (YAML)
│   └── 00001_class.yaml
├── CHANGELOG.md
└── README.md
```

Tests in `tests/` are described in YAML with multiple NL source blocks per file (see [docs/tests.md](docs/tests.md)). They are intended for use by a future compiler/tooling; the repo does not include a test runner.

## Using these specs

- **Implementing a compiler:** Use **specs.md** for syntax and semantics, **compiler.md** for all analyses and diagnostics to implement.
- **Implementing an interpreter or VM:** Use **specs.md**, **stdlib.md**, and **vm.md** together; **vm.md** defines the bytecode format and execution model so that different front-ends can target the same VM.
- **Writing NL code (once a compiler exists):** **specs.md** and **stdlib.md** are the programmer-facing references.

## Language highlights

- **Types:** `int`, `float`, `bool`, `byte`, `string` (characters as `string` of length 1), fixed-size arrays `T[]`, union types (`T|null`), `auto`, `typedef`.
- **OOP:** Classes, single inheritance, interfaces, visibility (`public` / `protected` / `private`), `abstract` / `final`, `readonly` classes/properties, `const` methods/parameters, constructors (`construct`) and destructors (`destruct`), `Cloneable` interface. Instance methods are virtual by default (Java-style).
- **Generics:** Template classes and methods with optional bounded type parameters (`extends`), monomorphization at compile time.
- **Control flow:** `if`/`else`, `while`, `for` (classic and for-each), `switch`/`match`, `break`/`continue`.
- **Functions:** Anonymous functions (closures), first-class callbacks, capture by reference with boxing when needed.
- **Exceptions:** Checked vs runtime exceptions, `throws`, `try`/`catch`/`finally`, stack traces.
- **Operators:** Arithmetic, comparison, logical, string concatenation, nullish coalescing (`??`, `?:`), overloaded operators.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and notable changes to the specification.
