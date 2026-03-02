# Changelog

## 0.7.0 — 2026-03-02

### Added

- **docs/stdlib.md** — § system.MapEntry&lt;K, V&gt;: new result type representing a key-value pair, with `K key` and `V value` fields. Used by `Map.entries()` and for-each iteration over maps.
- **docs/stdlib.md** — § system.Map: `keys()` returning `K[]`, `values()` returning `V[]`, `entries()` returning `MapEntry<K,V>[]`, and `forEach((K key, V value) => void f)` for callback-based iteration. Maps now support the for-each loop (`for (const auto entry : map)`). Iteration order is consistent across methods but implementation-defined (coherence § V-3).
- **docs/vm.md** — § For-each loops: added Map desugaring — compiler calls `entries()` then iterates the resulting array with an index-based loop.
- **docs/vm.md** — § Standard library binding: documented native dispatch for Map/List instance methods and `Map.forEach` closure invocation.

### Changed

- **docs/vm.md** — § Templates: `system.MapEntry<K,V>` added to the list of native template classes alongside List and Map.
- **docs/coherence.md** — V-3 resolved: Map iteration API fully specified.

## 0.6.0 — 2026-03-02

### Added

- **docs/specs.md** — § ValueEquatable interface: `valueEquals(const Self|null other)` and `valueHash()` for structural (value-based) equality of objects. Enables using objects as `system.Map` keys with value-based lookup (coherence § VI-3).
- **docs/specs.md** — § Comparison operators: clarified that `==` on references compares identity; value equality via ValueEquatable.

### Changed

- **docs/stdlib.md** — § system.Map: key equality semantics — primitives/string by value; reference types implementing ValueEquatable by valueEquals/valueHash; others by identity.
- **docs/vm.md** — § CMP_EQ: reference to ValueEquatable for value-based equality. § Templates: Map key lookup uses valueEquals/valueHash when K implements ValueEquatable.
- **README.md** — Language highlights: added ValueEquatable interface.
- **docs/coherence.md** — VI-3 resolved: valueEquals/valueHash convention specified.

## 0.5.0 — 2026-03-02

### Added

- **docs/specs.md** — § Bounded type parameters: `template <type T extends Bound>` syntax to constrain type parameters to a class or interface. Enables earlier compile-time errors and documentation of template contracts (coherence § VI-4).
- **docs/compiler.md** — E037: Type does not satisfy template bound. Template instantiation verifies bounded parameters at compile time.

### Changed

- **docs/compiler.md** — § Template instantiation: added bounded generics verification; concrete type must be subtype of bound.
- **docs/vm.md** — § Templates: noted that bounded constraints are compile-time only; no bound metadata in bytecode.
- **README.md** — Language highlights: Generics now mention bounded type parameters; error code count 36 → 37.
- **docs/coherence.md** — VI-4 resolved: bounded generics specified.
- **docs/milestones.md** — Type checking: added E037; error tests range E001–E037.

## 0.4.0 — 2026-03-02

### Added

- **docs/specs.md** — § Abstract classes and methods: abstract class, abstract method, rules, interaction with constructors.
- **docs/specs.md** — § Final classes and methods: final class (prevents inheritance), final method (prevents override).
- **docs/specs.md** — § Virtual method dispatch: all instance methods virtual by default (Java-style); no explicit `virtual` keyword.
- **docs/specs.md** — § Cloneable interface: `Self clone()` method, shallow copy by default; no dedicated `clone` keyword.
- **docs/compiler.md** — § Inheritance modifiers: error codes E032–E036 for abstract/final violations.

### Changed

- **README.md** — Language highlights: clarified that characters are represented as `string` of length 1 (no `char` type); added virtual-by-default for instance methods; renamed "`??` and `?:`" to "nullish coalescing (`??`, `?:`)" for clarity. Documentation table: stdlib description updated to `system.Env`, `system.ps.Process` (coherence V-5).
- **docs/specs.md** — Keywords: removed `virtual`, `delete`, `clone`; added links for `abstract`, `final`. Lifecycle: `new` only (no delete/clone).
- **docs/specs.md** — Destructor: wording corrected — "when the object becomes unreachable and is reclaimed by the garbage collector" (no explicit delete).
- **docs/specs.md** — Operators: removed `delete` and `clone` from the list.
- **docs/vm.md** — Replaced "Extensions (not yet specified)" with § Object lifecycle; removed provisional rules for virtual/abstract/final/clone/delete.
- **docs/vm.md** — Instance methods: updated to reflect specified abstract/final semantics.
- **docs/coherence.md** — V-2 resolved: all five keywords now specified or removed.

## 0.3.4 — 2026-03-02

### Fixed

- **docs/specs.md** — Exception inheritance rules: adopted Liskov-compatible rule from compiler.md (coherence § III-1). Child may declare `E` or a subclass of `E` for each parent exception; `throws IOException` alone is valid when parent has `throws Exception, IOException`. Updated examples for E016/E017.
- **docs/stdlib.md** — Arrays § built-in methods summary: added `forEach()`, `sort()`, `find()` to match specs.md and vm.md (coherence § III-2).
- **docs/specs.md** — Custom exception example: added `super(message);` as first statement in MyException constructor (coherence § IV-1).
- **docs/specs.md** — Entry point example: `argc < 3` → `argc < 4` since example accesses args[1], args[2], args[3] (coherence § IV-2).
- **docs/specs.md** — Anonymous function examples: parameter types `string` → `string|null` where null is checked (coherence § IV-3).
- **docs/specs.md** — Enum methods: translated `from()` / `tryFrom()` descriptions from French to English (coherence § IV-4).
- **docs/specs.md** — Removed undefined `char` from scalar types list in Parameter passing semantics (coherence § II-2). Documented that a character is represented as a `string` of length 1.

### Added

- **docs/specs.md** — § Planned: section for future spec features; `char` type listed as potentially added later.

### Declined

- **do-while loop** — NL will not implement `do-while`; `while` with `break` is sufficient (coherence § VI-1).
- **Multiple return values / tuples** — NL will not support tuples or multiple return values; use a custom class (coherence § VI-5).

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
