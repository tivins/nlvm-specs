# NL — Language Specification

**NL** is a statically-typed, object-oriented programming language designed for type safety and clarity. It draws from C++ (templates, type system), Java (syntax, file layout), modern PHP (match, typed enums), and Kotlin-style null safety with compile-time initialization guarantees.

**This repository contains only the specification** — language semantics, compiler checks, standard library API, and virtual machine. There is no compiler or interpreter implementation here.

---

## Documentation

| Document | Purpose |
|----------|---------|
| [specs.md](docs/specs.md) | Language syntax and semantics — *what* the language allows |
| [compiler.md](docs/compiler.md) | Semantic analyses, 42 error codes, 1 warning — *what* the compiler must verify |
| [stdlib.md](docs/stdlib.md) | Standard library API (`system`, `system.io`, etc.) — contract between user code and runtime |
| [vm.md](docs/vm.md) | Bytecode format, instruction set, execution model — *how* compiled code runs |
| [tests.md](docs/tests.md) | Test file format (YAML, `#NLFILE` blocks) for `tests/` |
| [milestones.md](docs/milestones.md) | Implementation roadmap — 9 phases from lexer to optimizations |
| [optimizations.md](docs/optimizations.md) | Optimization contract — what implementations *may* optimize |
| [showcase.md](docs/showcase.md) | Complete example program — NL principles in action |

---

## Where to start

| Goal | Path |
|------|------|
| **Discover NL** | [specs.md](docs/specs.md) → [showcase.md](docs/showcase.md) → Language highlights below |
| **Implement a compiler** | [specs.md](docs/specs.md) → [compiler.md](docs/compiler.md) → [milestones.md](docs/milestones.md) |
| **Implement a VM** | [specs.md](docs/specs.md) → [stdlib.md](docs/stdlib.md) → [vm.md](docs/vm.md) → [milestones.md](docs/milestones.md) |
| **Evaluate NL for SOLID/DDD** | [solid-compatibility.md](review/architecture/solid-compatibility.md), [ddd-compatibility.md](review/architecture/ddd-compatibility.md) |
| **Write or run tests** | [tests.md](docs/tests.md) → `tests/` |

---

## Project structure

```
nlvm-specs/
├── docs/       # All specification documents
├── review/     # Coherence tracker, security audit, architecture (SOLID, DDD), archives
├── tests/      # YAML acceptance tests (format in docs/tests.md)
├── CHANGELOG.md
└── README.md
```

The `tests/` folder holds YAML acceptance tests. They help implementers validate their compiler or VM against the specification. This repo does not include a test runner.

---

## Language highlights

- **Types:** `int`, `float`, `bool`, `byte`, `string`, fixed-size arrays `T[]`, union types `T|null`, `auto`, `typedef`
- **OOP:** Classes, single inheritance, interfaces, visibility, `abstract`/`final`, `readonly`/`const`, constructors/destructors
- **Generics:** Template classes and methods with bounded type parameters, monomorphization at compile time
- **Control flow:** `if`/`else`, `while`, `for` (classic and for-each), `switch`/`match`
- **Functions:** Anonymous functions (closures), first-class callbacks
- **Exceptions:** Checked vs runtime, `throws`, `try`/`catch`/`finally`
- **Operators:** Arithmetic, comparison, logical, string concatenation, nullish coalescing (`??`, `?:`), overloading

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and notable changes.

For the broader **language evolution** history (vvm repository, Nov–Dec 2025), see [vvm/CHANGELOG.md](https://github.com/tivins/vvm/blob/main/CHANGELOG.md).
