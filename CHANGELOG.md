# Changelog

## 0.3.0 — 2026-03-01

### Added

- **docs/vm.md** — NL Virtual Machine specification: execution model, bytecode format, instruction set
  (50+ opcodes), module binary format, object/array/string/enum/closure representation, method dispatch
  (vtable), exception handling (tables + stack unwinding), closure compilation, and compilation strategies
  for all major language features (templates, ref params, switch/match, ++/--, string concatenation,
  union types, operator overloading, nullish coalescing / elvis). Provisional rules documented for
  unspecified keywords (`virtual`, `abstract`, `final`, `clone`, `delete`).
- **docs/tests.md** — Test file format: YAML front matter (title, file_separator, expected_exit_code,
  expected_stdout, expected_stderr, compile_only), source blocks with separator line (`#NLFILE path`),
  multi-file layout and run vs compile-only semantics. README updated to reference tests.md.

## 0.2.0

### Added

- **docs/compiler.md** — Semantic analyses and compile-time guarantees (31 error codes, 1 warning).
- **docs/stdlib.md** — Standard library API for system interaction (I/O, net, threads, time, text, etc.).

## 0.1.0

### Added

- **docs/specs.md** — NL language specification (types, classes, enums, control flow, operators, exceptions,
  entry point).
- **tests/** — Initial test file structure.
