# Changelog

## 0.3.3 — 2026-03-01

### Added

- **docs/compiler.md** — § Compiler invocation (nlc): CLI specification for the compiler (arguments, options `-o`, `--entry`, `-c`, `--version`, `-h`, `-Werror`, `-v`, exit codes, conventions).
- **docs/vm.md** — § VM invocation (nlvm): CLI specification for the VM (arguments, options `--version`, `-h`, `-v`, `--module-path`, exit codes).
- **README.md** — Table links to compiler and VM CLI sections.
- **docs/milestones.md** — Summary table references to compiler.md § Compiler invocation (nlc) and vm.md § VM invocation (nlvm).

## 0.3.2 — 2026-03-01

### Added

- **docs/milestones.md** — Implementation roadmap: 8 milestones covering lexer/parser, semantic
  analysis, bytecode emission, VM core, objects/arrays/dispatch, exceptions/closures, standard
  library, and test runner integration. Each milestone lists scope, spec references, and testable
  deliverables. Includes a dependency graph. README updated to reference the new document.

## 0.3.1 — 2026-03-01

### Added

- **docs/coherence.md** — Coherence tracker listing all known inconsistencies, errors, and gaps across
  specs.md, stdlib.md, compiler.md, and vm.md. 20 items organized in 6 categories (syntax violations,
  nonexistent references, cross-document contradictions, incorrect examples, under-specified elements,
  missing features). Designed as a living checklist to track resolution progress.

### Fixed

- **docs/specs.md**, **docs/stdlib.md** — Resolved syntax violations (coherence § I): replaced nullable `?` suffix with explicit `|null` union in enum `tryFrom` signature, `FileHandle.readLine` and example, and `HttpResponse.headers` type to comply with the spec rule that `?` is not accepted.
- **docs/stdlib.md** — FileHandle example: `system.Out.writeLine(line)` → `system.Out.println(line)` (coherence § II-1).
- **docs/specs.md** — Enum example: replaced undefined `assert()` calls with explicit `if` checks (coherence § II-3).
- **docs/stdlib.md**, **docs/specs.md** — Naming coherence (V-5): `system.env` → `system.Env` (class); merged `system.ps` and `system.process` into namespace `system.ps` with class `system.ps.Process` (list, run, pid, getCwd, setCwd, exit), result types ProcessInfo and ProcessResult.
- **docs/specs.md** — Enum `from()`: exception type specified as `IllegalArgumentException` (coherence § V-1).

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
