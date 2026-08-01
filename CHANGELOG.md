# Changelog

## 0.8.49 — 2026-08-01

### Added

- **docs/optimizations.md** — § Observability: **Stack traces and frame-eliding optimizations** (new). The number and identity of the frames in `Exception.stackTrace` are **implementation-defined**, because inlining and tail call optimization remove frames the trace walk would otherwise see. Three invariants hold at every optimization level: the trace is never empty (it contains the `new` site), frames are ordered innermost first, and no frame may be fabricated for a call the program does not make. Reconstructing elided frames from inline metadata is quality-of-implementation, not conformance. Portable programs must not branch on `stackTrace.length()` or on a frame's `line`/`file` (spec issue [#1](https://github.com/nlvm-lang/nlvm-specs/issues/1)).
- **docs/optimizations.md** — § Observability: **Tail call optimization and `StackOverflowException`** (new). Unbounded tail-position recursion throws `StackOverflowException` without TCO and runs without stack growth with it; the divergence is permitted (maximum depth is already implementation-defined) and is the **only** sanctioned way for the set of exceptions thrown to differ between optimization levels. `StackOverflowException` is a resource-exhaustion signal, not a control-flow mechanism.
- **docs/optimizations.md** — § Prohibited transformations: item 5, **fabricating or reordering call frames** — optimizations may omit frames, never invent them or report them out of innermost-to-outermost order.
- **docs/optimizations.md** — § Testing: conformance tests must not assert an exact stack-trace frame count nor expect `StackOverflowException` from tail-position recursion.
- **tests/m6_0010_stacktrace_shape.yaml** (new): conformance test for the optimization-independent guarantee — an exception constructed directly and one returned by an inlinable static helper both carry a non-empty `stackTrace`; no frame count is asserted.

### Changed

- **docs/optimizations.md** — § Principles: principle 1 (*semantics preservation*) now defers to § Observability for what "observable" means; principle 4 (*portability*) states that a program depending on an implementation-defined value such as `stackTrace.length()` is not portable. This removes the contradiction between the permitted optimizations and the two principles.
- **docs/optimizations.md** — § Observability: the "Exceptions" bullet now covers exception types, order, throw point and catching handler, and excludes trace frames; "Call stack depth" added to the *not observable* list.
- **docs/optimizations.md** — § Compiler optimizations: the **Inlining** and **Tail call optimization** rows point at the new sections.
- **docs/vm.md** — § Call frame and operand stack: the **Stack overflow** note records that a runtime applying tail call optimization may never throw `StackOverflowException` for tail-position recursion, and should document that alongside its maximum call stack depth.
- **docs/vm.md** — § Stack trace construction: **Optimized builds** note — the walk only sees frames that exist; frame count and identity are implementation-defined, subject to the three invariants.
- **docs/specs.md** — § Exception class hierarchy: **Trace fidelity** note — `stackTrace` is a diagnostic view, not a guaranteed value; do not branch on frame count or on a frame's `line`/`file`.
- **docs/stdlib.md** — § Exceptions: `StackOverflowException` row now states it is not a control-flow mechanism and may never be thrown under tail call optimization.

### Updated references

- **docs/milestones.md** — M6 testable: stack-trace tests assert frame presence and order, not an exact count. M9 testable: regression tests must not assert frame counts or `StackOverflowException` from tail-position recursion.
- **review/coherence.md** — new section *Items raised after the 2026-03-03 audit*, with item **P-1** (this resolution) closed in 0.8.49.

## 0.8.48 — 2026-07-30

### Added

- **docs/stdlib.md** — § system.thread.Thread: `void interrupt()` and `bool isInterrupted()`. `interrupt()` sets a per-thread interrupt flag; `isInterrupted()` reads it without clearing. Until now `InterruptedException` was declared by `join()`/`join(int)`/`sleep(int)` with no API able to raise it — nothing in the language could interrupt a thread, so the exception was unreachable by construction (nlvm issue [#4](https://github.com/nlvm-lang/nlvm/issues/4)).
- **docs/stdlib.md** — § system.thread.Thread: **Interruption** note defining the semantics of the flag: `join()`/`join(int)`/`sleep(int)` are the only interruption points; a thread blocked at one wakes up and throws promptly (`join(int)` reports an interruption by throwing, never by returning `false`); delivery clears the flag; the flag is otherwise sticky (interrupting an unstarted, finished or non-blocking thread is well-defined and never lost); `Mutex.lock()`, `Semaphore.acquire()` and blocking `system.io`/`system.net` operations are *not* interruption points; the thread running `main` has no `Thread` object and therefore cannot be interrupted.
- **tests/m7_0050_thread_interrupt.yaml** (new): conformance test for interruption — a worker blocked in `Thread.sleep(60000)` is interrupted, catches `InterruptedException`, and reports `isInterrupted() == false` afterwards (delivery clears the flag).
- **docs/vm.md** — § Threading model: interruption requirements on the VM — per-thread interrupt flag, visible to the target without program-level synchronization, prompt wake-up of a thread already blocked at an interruption point, and no forced thread termination.

### Changed

- **docs/stdlib.md** — § Exceptions: `InterruptedException`'s description now points at the Interruption note instead of describing the trigger inline.

## 0.8.47 — 2026-07-21

### Added

- **docs/specs.md** — § Exception class hierarchy: `Exception.printStackTrace()` — writes `message` and one `"    at " + file + ":" + line` line per `stackTrace` frame to `system.Err`, mirroring the VM's own unhandled-exception output. Documented limitation: without a reflection API, the output cannot be prefixed with the exception's runtime class name (unlike Java's `Throwable.printStackTrace()`).
- **docs/specs.md** — § Planned: **Reflection API** — no `getClass()`/runtime-type-name mechanism exists yet; tracked as a prerequisite for extending `printStackTrace()` (or a future `toString()`) with the exception's runtime class name.

## 0.8.46 — 2026-07-21

### Added

- **docs/stdlib.md** — § system.db (new): SQL database connectivity. Driver-agnostic base types `Connection` (opaque, returned by driver factories), `PreparedStatement` (0-based `?` placeholder binding via `bindInt`/`bindFloat`/`bindBool`/`bindString`/`bindBytes`/`bindNull`), `ResultSet` (cursor with `Row|null next()`, for-each iterable), `Row` (typed accessors `getInt`/`getFloat`/`getBool`/`getString`/`getBytes` returning `T|null` for SQL NULL, by 0-based index or column name), and enum `ColumnType` (`Integer`, `Float`, `Text`, `Blob`, `Bool`, `Null`). Transaction API on `Connection`: `beginTransaction`/`commit`/`rollback`. `Connection.lastInsertId` for auto-increment retrieval. Convenience `query(string)` / `execute(string)` for constant SQL (no user input). Resource semantics parallel to `FileHandle`/`TcpStream` (idempotent `close()`, use-after-close throws `SqlException`).
- **docs/stdlib.md** — § system.db.sqlite (new): SQLite driver. `Sqlite.open(path)` / `Sqlite.open(path, SqliteOpenMode)` / `Sqlite.openMemory()`; enum `SqliteOpenMode` (`ReadOnly`, `ReadWrite`, `ReadWriteCreate`). Path-traversal security note aligned with `system.io.File`.
- **docs/stdlib.md** — § system.db.mysql (new): MySQL / MariaDB driver. `Mysql.connect(MysqlConfig)`; `MysqlConfig` fields `host`, `port` (default 3306), `user`, `password`, `database` (default `""`), `useTls` (default `true`). Mandatory TLS certificate validation when `useTls = true` (parallel to `system.net.Http` for `https://`); credentials-handling security note.
- **docs/stdlib.md** — § Exceptions: `SqlException` (checked, `system.db`) — thrown by all `system.db.*` operations on connection error, TLS validation failure, SQL error, constraint violation, bind/column resolution failure, or use of a closed/stale handle; carries `sqlState` (SQLSTATE, empty when unavailable) and `errorCode` (driver-specific, `0` when unavailable).

### Changed

- **docs/specs.md** — § Exception class hierarchy: added `SqlException extends Exception` with public `sqlState: string` and `errorCode: int`.

### Updated references

- **docs/milestones.md** — M7 scope: added Database bullet (drivers, opaque `Connection`, `?` placeholders, 0-based binding).

## 0.8.45 — 2026-07-20

### Added

- **docs/stdlib.md** — § system.text.json (new): JSON (RFC 8259) parsing and serialization. `JsonValue` abstract base with `JsonNull`, `JsonBool`, `JsonNumber`, `JsonString`, `JsonArray`, `JsonObject` subclasses (type predicates `isX()`, typed accessors `asX()` throwing `InvalidCastException` on mismatch); `Json.parse`/`tryParse`/`stringify` static entry points. `JsonObject.get` distinguishes an absent key (`null`) from a present key with JSON `null` value (`JsonNull`).
- **docs/stdlib.md** — § Exceptions: `JsonFormatException` (checked, `system.text.json`) — thrown by `Json.parse` on malformed input; carries `line`, `column`, `expectedToken`, `foundToken` for precise error reporting.

### Changed

- **docs/specs.md** — § Exception class hierarchy: added `JsonFormatException extends FormatException`.

## 0.8.44 — 2026-07-18

### Changed

- **docs/compiler.md** — § Union type compatibility: clarified that a value is assignable to a union constituent
  "or a subtype thereof" per the [implicit conversions](docs/specs.md#type-conversions-and-casting) table —
  explicitly includes classes implementing an interface constituent (previously worded as "subclass thereof",
  ambiguous as to whether interface implementation counted). Added an `Animal|null pet = new Dog()` example.
- **tests/m5_0020_instanceof.yaml** — `Animal animal = dog;` changed to `Animal|null animal = dog;`; the original
  non-nullable declaration made the later `animal = null;` a legitimate E003 violation, contradicting the test's
  `expected_exit_code: 0`.

## 0.8.43 — 2026-07-14

Security hardening pass — resolves coherence VIII-2, VIII-3, VIII-4, VIII-5, VIII-6, VIII-7, VIII-9, VIII-10
(security-audit SEC-02, SEC-03, SEC-05, SEC-06 (Env), SEC-08, SEC-09, SEC-10, SEC-16, SEC-18, SEC-23).

### Added

- **docs/stdlib.md** — § system.Int / system.Float / system.Bool: `tryParse(string) : T|null` — safe-by-default parsing, returns `null` instead of throwing on invalid input (naming aligned with `enum.tryFrom`); safety note on `parse` with untrusted input.
- **docs/stdlib.md** — § system.SecureRandom (new): OS-backed CSPRNG (`nextBytes`, `nextInt`, bias-free bounded `nextInt`), not seedable. § system.Uuid: `random()` specified as **UUID v4** generated from the CSPRNG. § system.Random: documented as unsuitable for security purposes.
- **docs/stdlib.md** — § system.text.Regex: `escape(string)` for literal use of user input in patterns; `match`/`matchFirst` specified as **partial match** (anchor with `^…$` for full match, consistent with Grep).
- **docs/vm.md** — § Module integrity (new): integrity trailer at end of module (`hash_algo` u8 + hash; SHA-256 recommended by default), mandatory load-time verification when present, trust model documented. Module format version 1 → 2.
- **tests/m7_0040_tryparse_null.yaml** — Run test for `Int.tryParse` / `Float.tryParse` (null on invalid input).

### Changed

- **docs/specs.md** — § Native types: new **Integer overflow** subsection — `int` is 64-bit two's complement, `+ - *` wrap silently, comparator anti-pattern `a - b` documented; all comparator examples now use the overflow-safe `a <=> b`. § Entry point: Calculator example rewritten with `tryParse` + null check.
- **docs/stdlib.md** — § system.io.File: **path traversal** security note (no sanitization; validation is the caller's responsibility; `Path.normalize` + base-directory check for untrusted input). § system.io.FileHandle / system.net.TcpStream: byte-array `read`/`write` must throw `IndexOutOfBoundsException` on `offset < 0 || length < 0 || offset + length > buffer.length()`, checked before any I/O and immune to integer overflow; `UdpSocket.receive` truncates oversized datagrams. § system.net.Http: mandatory TLS certificate validation for `https://` (chain, expiration, hostname; failure throws IOException). § system.Env: thread-safety (concurrent set/remove is UB — synchronize) and security note (secrets exposure; `PATH`/`LD_PRELOAD` injection with `Process.run`). § Exceptions table: bounds violations added to `IndexOutOfBoundsException`.

### Updated references

- **docs/milestones.md** — M3: line-number table + module integrity; M7: SecureRandom.
- **review/security-audit.md** — Resolution notes on SEC-02, SEC-03, SEC-05, SEC-06, SEC-08, SEC-09, SEC-10, SEC-16, SEC-17, SEC-18, SEC-22, SEC-23; prioritized recommendations updated (all "Immediate" items now resolved).
- **review/coherence.md** — VIII-2, VIII-3, VIII-4, VIII-5, VIII-6, VIII-7, VIII-9, VIII-10 resolved; **all items closed**, archived to `archives/coherence_closed_20260714.md`.

## 0.8.42 — 2026-07-14

VM and module format gaps — resolves coherence IV-1, IV-2, IV-3, IV-4, IV-7 (security-audit SEC-17).

### Added

- **docs/vm.md** — § Module format: class flag bits 3 `ABSTRACT`, 4 `FINAL` (VM rejects `NEW` on abstract classes and extension of final classes at link time; both bits set is a load error). § Method descriptor: method flag bits 8 `ABSTRACT`, 9 `FINAL` (link-time rejection of final-method overrides).
- **docs/vm.md** — § Method descriptor: **line-number table** (`line_table_count` u16 + `{start_pc: u16, line: u32}` entries) for stack-trace debug info; `line_table_count = 0` for stripped builds (stack trace lines report `0`). Module format version bumped (see 0.8.43 — v2 covers both changes).
- **docs/vm.md** — § Method descriptor: **abstract method representation** — ABSTRACT methods have `code_length = 0`, empty code/exception/line tables; loader rejects abstract-with-code and concrete-without-code.
- **docs/vm.md** — § Control flow: documented wide-conditional-branch strategy — no `IF_TRUE_W`/`IF_FALSE_W`; the compiler emits the inverted condition over a `GOTO_W` for targets beyond ±32 KiB.

### Changed

- **docs/vm.md** — § Stack trace construction: `stackTrace` is captured and assigned **during the base `Exception` constructor** (native capture, exception-constructor frames excluded) — an ordinary readonly assignment inside `construct`; the VM needs **no readonly bypass**. `line` derived from the new line-number table.
- **docs/specs.md** — § Exception class hierarchy: stack trace capture note aligned with vm.md.

### Updated references

- **review/coherence.md** — IV-1, IV-2, IV-3, IV-4, IV-7 resolved.

## 0.8.41 — 2026-07-14

Language decisions — resolves coherence II-2, II-8, II-9, II-11, II-16, VI-1 through VI-8. Error codes 44 → 49
(E045–E049).

### Added

- **docs/specs.md** — § Operator precedence: full 12-level precedence table (primary/postfix → unary → multiplicative → additive → `<=>` → relational/`instanceof` → equality → `&&` → `||` → ternary → `??`/`?:` → assignment) with associativity.
- **docs/specs.md** — § Interface inheritance: `interface A extends B, C` — any number of parent interfaces; implementing classes implement all inherited methods; parents are supertypes for upcasts and `instanceof`; conflicting diamond declarations → E041.
- **docs/specs.md** — § Constructor chaining (`this(...)`): delegation to a same-class constructor; at most one delegation call (`this` or `super`), first statement (E045); acyclic chains (E046); definite-assignment credit from the target constructor.
- **docs/specs.md** — § `Self` in interfaces: `Self` denotes the implementing class; signatures instantiated per implementing class at compile time (covariant returns for `Cloneable.clone`, parameter types for `ValueEquatable.valueEquals`); inherited implementations keep the defining class's resolution.
- **docs/specs.md** — § Match exhaustiveness: `match` must be exhaustive at compile time (E047) — enum: all cases or `default`; `bool`: both values or `default`; int/string/others: `default` required; duplicate arms unreachable; `default` last. A `match` never fails at runtime.
- **docs/specs.md** — § Common result type: shared 5-rule algorithm (identity, implicit conversion, `null` literal, nearest common ancestor, otherwise **union**) defining the result types of `? :`, `??`, `?:`, and lambda return deduction.
- **docs/specs.md** — § Return type deduction rules (anonymous functions): single-expression body; common result type of `return` expressions for block bodies; `void` when no valued return; mixing bare/valued returns is an error; target typing takes precedence.
- **docs/specs.md** — § Overloadable operators: exhaustive table with required signatures (`+ - * / %` → `type … const`; `+= …` → `Self`; `< > <= >=` → `bool … const`; `<=>` → `int … const`; unary `-`/`!` const; `++`/`--` → `Self`, same method prefix/postfix — postfix yields the mutated reference) and the non-overloadable list (`==`/`!=`, `&&`/`||`, `=`, conditionals, `.`/`new`/cast/`instanceof`, `[]` moved to Planned).
- **docs/specs.md** — § Exception handling: catch clause ordering — clauses tested in order; unreachable clause (same type or supertype caught earlier) → E048.
- **docs/specs.md** — § Readonly / § Final: modifier combinations — `readonly` orthogonal to `abstract`/`final` (`abstract class readonly`, `final class readonly` valid); `abstract` + `final` mutually exclusive on classes too (E049); canonical order `[abstract | final] class [readonly] Name`.
- **docs/compiler.md** — § Type narrowing (smart casts): flow-sensitive narrowing for locals and parameters (`!= null`, `instanceof`, early exits, `&&`/`||` chains, ternary conditions); invalidation on reassignment; no narrowing for closure-mutated captures or properties (copy to a local). § Match exhaustiveness (E047), § Unreachable catch clauses (E048), § Constructor delegation (E045, E046), § Final: conflicting modifiers (E049). Error count 44 → 49.
- **docs/vm.md** — § Constructors: `this(...)` delegation compiles to `INVOKE_SPECIAL` on the sibling constructor. § Match expressions: no runtime unmatched path (compile-time exhaustiveness). § Module format: INTERFACE modules use the `interfaces` list for extended interfaces.
- **tests/m2_0040_compile_e047_match_not_exhaustive.yaml**, **tests/m2_0050_compile_e048_unreachable_catch.yaml**, **tests/m2_0060_smart_cast_narrowing.yaml**, **tests/m5_0030_constructor_chaining.yaml** — new tests.

### Updated references

- **docs/milestones.md** — Error count 44 → 49; M2 scope: type narrowing, match exhaustiveness (E047), unreachable catch (E048), constructor delegation (E045–E046), conflicting modifiers (E049); error test range E001–E049.
- **README.md** — Error code count 44 → 49.
- **review/coherence.md** — II-2, II-8, II-9, II-11, II-16, VI-1…VI-8 resolved.

## 0.8.40 — 2026-03-06

### Changed

- **docs/specs.md** — § Stringable interface: `toString()` is now `toString() const` to enforce read-only semantics. Explicit note that the interface does not declare `throws`; implementations may throw runtime exceptions if needed. All examples updated (Person, Bar, Point).
- **docs/stdlib.md** — § Core interfaces: Stringable API updated to `string toString() const`; added note on no `throws`.
- **docs/compiler.md** — § Const methods: when implementing an interface method declared as `const`, the implementing method must be `const`. New E044 for non-const implementation of const interface method. Error count 43 → 44.

### Updated references

- **docs/milestones.md** — Error count 43 → 44; error test range E001–E044.
- **README.md** — Error code count 43 → 44.
- **.cursor/skills/nl-specs/** — Error code range E001–E044.
- **docs/showcase.md** — Task.toString() updated to `toString() const`.
- **docs/milestones.md** — § Milestone 2: added E044 to Immutability enforcement scope.
- **tests/m2_0030_compile_e044.yaml** — Compile-error test for E044 (non-const toString() implementing Stringable).

## 0.8.39 — 2026-03-06

### Added

- **Docsify front** — `index.html`, `_sidebar.md`, `.nojekyll` for a browsable docs site. Serves specs from markdown with sidebar navigation and search. Deploy via GitHub Pages (see README).
- **Docsify dark/light theme** — `docsify-darklight-theme` with toggle, system preference detection, and localStorage persistence.

## 0.8.38 — 2026-03-06

### Changed

- **docs/stdlib.md** — § system.ps.Process: added security warning for `run(string command)` — never interpolate user-controlled input; prefer `run(string[] args)` to bypass shell interpretation. Added safe example in code block. Resolves coherence VIII-1.

### Updated references

- **review/coherence.md** — VIII-1 marked resolved.

## 0.8.37 — 2026-03-05

### Added

- **docs/specs.md** — § Keywords: added `instanceof`. § Other operators: `expr instanceof ClassName` — runtime type test returning `bool`; left operand must be a reference type, right operand must be a class or interface; `null` produces `false`. Compiles to `INSTANCEOF` opcode.
- **docs/compiler.md** — § Instanceof expression: validation rules for left operand (reference type) and right operand (class or interface).
- **tests/m5_0020_instanceof.yaml** — Run test for `instanceof` with class, interface, and null.

### Updated references

- **review/coherence.md** — II-1 marked resolved.

## 0.8.36 — 2026-03-05

### Changed

- **review/coherence.md** — Archived 49 resolved items to `archives/coherence_closed_20260305.md`; main file now lists only 28 open items (II, IV, VI, VIII) for clarity.

## 0.8.35 — 2026-03-05

### Added

- **docs/stdlib.md** — § system.io.FileMode: enum with cases `Read`, `Write`, `Append`, `ReadWrite`, `ReadWriteTruncate`, `ReadWriteAppend` controlling how a file is opened. § system.io.File: overload `open(string path, FileMode mode)` for mode-controlled access. Resolves coherence V-6.

### Updated references

- **docs/specs.md** — § Standard library: added FileMode reference and open(path, mode) description.
- **docs/milestones.md** — File system: added FileMode to scope.
- **review/coherence.md** — V-6 marked resolved.

## 0.8.34 — 2026-03-05

### Changed

- **docs/stdlib.md** — § system.List, § system.Map: explicit **thread safety** note — not thread-safe; heap objects are shared across threads (vm.md § Threading model); caller must use `system.thread.Mutex` (or another synchronization primitive) when multiple threads access the same list or map. Resolves coherence V-5.

### Updated references

- **review/coherence.md** — V-5 marked resolved.

## 0.8.33 — 2026-03-05

### Added

- **docs/stdlib.md** — § system.thread.Thread: `isAlive()` returns `true` if the thread has been started and has not yet finished, `false` otherwise. Non-blocking. Resolves coherence V-4.

### Updated references

- **review/coherence.md** — V-4 marked resolved.

## 0.8.32 — 2026-03-05

### Added

- **docs/compiler.md** — § Import name resolution: E043 when an import would create a duplicate unqualified name (conflict with class in file, type in namespace, or another import). Resolves coherence IV-8.

### Changed

- **docs/specs.md** — § Import rules: explicit "no duplicate unqualified names" rule; conflict must be rejected; use `as` or fully qualified name to resolve. Cross-reference to compiler.md § Import name resolution.

### Updated references

- **review/coherence.md** — IV-8 added and marked resolved.
- **docs/milestones.md** — Error count 42 → 43; added Import name resolution (E043) to Milestone 2 scope; error test range E001–E043.
- **README.md** — Error code count 42 → 43.
- **.cursor/skills/nl-specs/** — Error code range updated to E001–E043.

## 0.8.31 — 2026-03-05

### Added

- **docs/stdlib.md** — § system.List: `remove(int index)` removes and returns the element at index; `contains(T value)` checks presence (value equality for primitives/string, ValueEquatable for refs, else reference identity). Resolves coherence V-3.
- **docs/vm.md** — § Native template classes: added `remove`, `contains` to List instance methods list.

## 0.8.30 — 2026-03-05

### Changed

- **docs/stdlib.md** — § system.String: `trim` and `split` are now instance methods on string values (`text.trim()`, `s.split(delimiter)`); static forms `trim(string)` and `split(string, delimiter)` kept for flexibility. Resolves coherence V-2.
- **docs/showcase.md**, **docs/specs.md** — Updated examples to use instance method syntax.

## 0.8.29 — 2026-03-05

### Changed

- **docs/stdlib.md** — Renamed `parseInt`, `parseFloat`, `parseBool` to `parse` for unified API across `system.Int`, `system.Float`, and `system.Bool`.
- **docs/specs.md** — § Planned: added **Parsable interface** — template interface with static abstract `parse(string)` for generic parsing; requires static abstract interface members support.

### Updated references

- **docs/specs.md**, **docs/showcase.md**, **docs/compiler.md**, **review/coherence.md**, **review/security-audit.md** — All parse method references updated.

## 0.8.28 — 2026-03-05

### Added

- **docs/stdlib.md** — § system.Bool: `parse(string s) throws IllegalArgumentException` for converting `"true"` / `"false"` to `bool`. Resolves coherence V-1. *(Renamed to `parse` in 0.8.29.)*

## 0.8.27 — 2026-03-05

### Changed

- **docs/specs.md** — § Custom methods and properties: enums may declare custom static methods, instance methods, and static properties beyond built-in `from()`, `tryFrom()`, and `value`. Style recommendation to keep enums lightweight. Resolves coherence II-10.
- **docs/vm.md** — § Enum representation: added note that custom methods and properties are compiled like class members.
- **review/coherence.md** — Marked II-10 as resolved.

## 0.8.26 — 2026-03-05

### Added

- **docs/showcase.md** — Complete example program (task manager CLI) illustrating NL's core principles: null safety, readonly classes, checked exceptions, typed enums, templates, interfaces, closures, and standard library usage. Linked from README.

## 0.8.25 — 2026-03-04

### Changed

- **docs/compiler.md** — § Exception inheritance rules: clarified that runtime exceptions in documentation-only `throws` are not subject to E016/E017 inheritance rules; overrides may freely add or remove them.
- **docs/compiler.md** — § Duplicate definitions: clarified that signature = name + parameter types only; return type and `throws` do not distinguish methods for E041.
- **review/architecture/solid-compatibility.md** — § LSP: added runtime exception `throws` interaction analysis (neutral for LSP). § ISP: updated to reflect multiple interface implementation now formally defined (coherence II-7 resolved). Summary scores and points-to-watch updated.

## 0.8.24 — 2026-03-04

### Changed

- **docs/compiler.md** — § Checked exception propagation: `throws` may list runtime exceptions for documentation purposes; compiler does not enforce them. Resolves coherence I-5.
- **docs/specs.md** — § Float literal format: defined accepted float literal formats (`3.14`, `.5`, `2.`, `0.0`); no scientific notation. Resolves coherence II-5.
- **docs/specs.md** — § Extends, Implements: defined comma-separated syntax for multiple interfaces (`implements A, B, C`) with example. Resolves coherence II-7.
- **docs/specs.md** — § Static methods: explicitly forbids `this`, `Self`, and instance member access in static context.
- **docs/compiler.md** — § Static context restrictions: added E040. Resolves coherence II-12.
- **docs/specs.md** — § Source code files: explicitly states nested class definitions are not allowed. Resolves coherence II-13.
- **docs/specs.md** — § Loops: multiple same-type init declarations allowed in for-loop; variables scoped to the for block. Resolves coherence II-14 and II-15.
- **docs/compiler.md** — § Duplicate definitions: added E041 (duplicate method signature) and E042 (duplicate class definition). Resolves coherence II-17 and IV-6.
- **docs/specs.md** — § Elvis operator: examples rewritten with same-type operands; type compatibility note added. Resolves coherence III-7.
- **docs/specs.md** — § Type conversions: `float → int` overflow behavior changed from "undefined" to clamping (`INT_MIN`/`INT_MAX`), aligned with vm.md. Resolves coherence IV-5.
- **docs/compiler.md** — § Terminal statements: `return`, `throw`, and `Process.exit()` defined as terminal statements (unreachable code after). Resolves coherence VI-9.
- **docs/stdlib.md** — § system.ps.Process: `exit` marked as terminal statement.
- **docs/specs.md** — § Exception class hierarchy: added `StackOverflowException extends RuntimeException`.
- **docs/vm.md** — § Call frame: stack overflow behavior defined (throws `StackOverflowException`).
- **docs/stdlib.md** — Exceptions table: added `StackOverflowException`. Resolves coherence V-8.
- **review/coherence.md** — Marked 11 items as resolved (I-5, II-5, II-7, II-12, II-13, II-14, II-15, II-17, III-7, IV-5, IV-6, V-8, VI-9); updated summary counts.

## 0.8.23 — 2026-03-04

### Added

- **tests/m7_0030_read_after_close_cwe416.yaml** — Run test: read after close on FileHandle must throw IOException (CWE-416).

### Changed

- **review/security-audit.md** — Added note: each finding must have a corresponding test in `tests/` per [tests.md](docs/tests.md), with CWE in the file name for traceability.
- **docs/stdlib.md** — § system.io.FileHandle, system.net.TcpStream, system.net.UdpSocket: specified that read/write/flush (FileHandle), read/write (TcpStream), send/receive (UdpSocket) on a closed handle throw `IOException`. Resolves SEC-11 (Read/Write After Close).

## 0.8.22 — 2026-03-04

### Added

- **review/security-audit.md** — Security audit: 26 findings across 5 severity levels (3 Critical, 7 High, 8 Medium, 5 Low, 3 Informational). Covers command injection (CWE-78), path traversal (CWE-22), unsigned bytecode (CWE-494), denial of service via resource exhaustion (CWE-400), integer overflow (CWE-190), race conditions (CWE-362), missing TLS validation (CWE-295), weak randomness (CWE-338), SSRF (CWE-918), and more. Each finding mapped to CWE identifiers and known CVE patterns.
- **review/coherence.md** — § VIII: 10 security-related specification gaps tracking the most impactful findings from the security audit (VIII-1 through VIII-10).

## 0.8.21 — 2026-03-04

### Changed

- **docs/specs.md** — § Variable shadowing: local/parameter may shadow class field (use `this.name` for field); inner block variable may shadow outer. Block-scoping clarified. Resolves coherence II-3.
- **docs/compiler.md** — § Variable shadowing: cross-reference to specs; shadowing allowed, no error emitted.

## 0.8.20 — 2026-03-04

### Changed

- **docs/specs.md** — § Const methods and parameters: local variables may be declared `const`; they cannot be reassigned after initial assignment. Resolves coherence II-6.
- **docs/compiler.md** — § Const local variables: documents E012 for assignment to const locals; generalized E012 message to "const variable".
- **docs/milestones.md** — Milestone 2: "const parameters" → "const parameters and locals" (E012).

## 0.8.19 — 2026-03-04

### Changed

- **docs/specs.md** — § Native types: explicitly documents that byte literals are not supported; use `(byte) intExpr` to obtain byte values. Resolves coherence II-4.

## 0.8.18 — 2026-03-04

### Changed

- **README.md** — Added "Evaluate NL for SOLID/DDD" to "Where to start" table; updated project structure to mention architecture analyses (SOLID, DDD) in `review/`.

## 0.8.17 — 2026-03-03

### Changed

- **review/architecture/solid-compatibility.md** — Translated from French to English.
- **review/architecture/ddd-compatibility.md** — Translated from French to English.

## 0.8.16 — 2026-03-03

### Added

- **review/architecture/solid-compatibility.md** — NL compatibility analysis with SOLID principles.

## 0.8.15 — 2026-03-03

### Added

- **review/architecture/ddd-compatibility.md** — NL compatibility analysis with Domain-Driven Design (DDD) principles.

## 0.8.14 — 2026-03-03

### Changed

- **README.md** — Added link to [vvm/CHANGELOG.md](https://github.com/tivins/vvm/blob/main/CHANGELOG.md) for broader language evolution history (Nov–Dec 2025).

## 0.8.13 — 2026-03-03

### Added

- **docs/optimizations.md** — § Compiler optimizations: incremental compilation — cache compiled modules per source file, recompile only modified files and dependents; uses module-per-file model and explicit `use` dependencies.
- **docs/compiler.md** — § Compiler invocation: `--incremental`, `-i` option for incremental compilation.
- **docs/milestones.md** — Milestone 9: added incremental compilation to compiler optimizations scope.

## 0.8.12 — 2026-03-03

### Changed

- **README.md** — Restructured for clarity: single documentation table, "Where to start" by audience (discover / implement compiler / implement VM / tests), condensed language highlights, removed redundancies. Clarified that `tests/` help implementers validate their compiler or VM against the spec.

## 0.8.11 — 2026-03-03

### Changed

- **README.md** — Simplified project structure: removed per-file listing of docs and tests; kept folder-level overview with pointer to docs/tests.md. Moved reading-order paragraph above the doc list, converted doc names to links, removed bold.

## 0.8.10 — 2026-03-03

### Added

- **tests/m1_0010_parse_minimal.yaml** — Parse minimal (namespace, class, main); compile_only.
- **tests/m2_0010_compile_valid.yaml** — Compile valid simple program; compile_only.
- **tests/m2_0020_compile_e003.yaml** — Compile error E003 (null to non-nullable).
- **tests/m4_0020_arithmetic_int.yaml** — Run test: return 2 + 3 → exit 5.
- **docs/tests.md** — `expected_compile_error` header key for compile-fail tests.

## 0.8.9 — 2026-03-03

### Added

- **docs/tests.md** — § File naming: pattern `m{N}_{XXXX}_{name}.yaml` (milestone, index with spacing, descriptive name).
- **tests/m4_0010_minimal_main.yaml** — First run test: entry point only, main returns 0 (Milestone 4).

### Changed

- **tests/** — Renamed `00001_class.yaml` to `m5_0010_class_instantiation.yaml`; updated README project structure.

## 0.8.8 — 2026-03-03

### Added

- **.cursor/skills/nl-specs/** — Cursor skill for NL specification workflows: writing/editing specs, coherence checks (review/coherence.md), YAML test creation, cross-document consistency. Includes SKILL.md and reference.md.

## 0.8.7 — 2026-03-03

### Changed

- **Project structure** — Created `review/` folder; moved `archives/` into `review/archives/`; moved `docs/coherence.md` to `review/coherence.md`. Updated README and internal links.

## 0.8.6 — 2026-03-03

### Added

- **docs/optimizations.md** — Optimization contract: principles (semantics preservation, side-effect ordering), compiler optimizations (constant folding, dead code elimination, devirtualization, inlining, tail call, string literal concatenation), VM optimizations (string interning, JIT, superinstructions, inline caching, GC tuning), prohibited transformations, and observability definition.
- **docs/compiler.md** — § Optimizations: link to optimizations.md.
- **docs/vm.md** — Links to optimizations.md from string interning and devirtualization sections.
- **docs/milestones.md** — Milestone 9: Optimizations (optional phase after test runner).
- **README.md** — Added optimizations.md to documentation table and project structure.

### Changed

- **docs/vm.md** — Summary: added note pointing to optimizations.md for optimization-related guarantees.
- **docs/coherence.md** — Added optimizations.md to the list of tracked specification documents.

## 0.8.5 — 2026-03-03

### Changed

- **docs/specs.md** — § Entry point: removed redundant `argc` parameter from `main` signature. Use `args.length()` instead. New signature: `public static int main(string[] args)`.
- **docs/vm.md** — § Program startup: build only `args`; invoke `main(string[])` with `args` in local 0.
- **docs/compiler.md** — § Entry point validation: expected signature `main(string[])`; E029 message updated.
- **docs/tests.md**, **docs/milestones.md** — Updated main signature references.
- **tests/00001_class.yaml** — Main method uses new signature.

## 0.8.4 — 2026-03-03

### Added

- **docs/specs.md** — § Loops: `const` is **optional** in for-each; both `for (auto item : collection)` and `for (const auto item : collection)` are valid. **Copy semantics**: loop variable holds a copy of each element (value copy for value types, reference copy for reference types). **Implicit const in const context**: when iterating over `this.property` in a const method, or over a const/const ref parameter, the loop variable is implicitly read-only (deep immutability).
- **docs/compiler.md** — § For-each loop in const context: E039 — Cannot modify loop variable when iterating over read-only collection.

### Changed

- **docs/compiler.md** — Auto type deduction: loop variables now support both forms (with or without `const`).
- **docs/vm.md** — § For-each loops: clarified that both forms desugar identically; `const` affects only compile-time checks. Noted that `ARRAY_LOAD` + `STORE` yields copy semantics (value or reference copy per element type).
- **docs/milestones.md** — Immutability enforcement: added E039; error count 38 → 39.
- **docs/coherence.md** — I-3, III-9, III-10 resolved: for-each const optional, stdlib examples valid.

## 0.8.3 — 2026-03-03

### Fixed

- **docs/specs.md** — § Template class: duplicate variable `v1` → `v2` in usage example (coherence III-3).
- **docs/specs.md** — § Fluent methods: added missing `return this;` in `save()` method (coherence III-8).
- **docs/specs.md** — § Enums: "tailing coma" → "trailing comma" (coherence VII-3).
- **docs/specs.md** — § Nullish coalescing: examples used non-nullable `MyObject` with `null` assignment (E003); changed to `string|null` for type consistency (coherence III-4, III-5).
- **docs/specs.md** — § Elvis operator: example used non-nullable `MyObject` with `null` assignment (E003); changed to `string|null` (coherence III-6).
- **docs/specs.md** — § Exception class hierarchy: `IllegalArgumentException` comment now mentions `enum.from()` in addition to `TimeZone.get` (coherence I-6).
- **docs/stdlib.md** — Introductory text: removed `system.Env` from the namespace list (it is a class in `system`, not a namespace) (coherence I-4).
- **docs/stdlib.md** — Namespace table: added `Env` and core interfaces to `system` namespace description.
- **docs/stdlib.md** — § system.Out: `print`/`println` overloads for `int`, `float`, `bool` changed from "may" to **"must"** be provided (coherence I-7).
- **docs/stdlib.md** — § system.String example: `length()` result 15 → **16** (coherence III-1); `substring(2, 8)` result `"Hello"` → **`"Hello,"`** (coherence III-2).
- **docs/stdlib.md** — § system.io.Path example: removed extraneous space in array initializer `new string[] {…}` → `new string[]{…}` (coherence III-11).
- **docs/stdlib.md** — Exceptions table: `IllegalArgumentException` namespace changed from `system.time` to **`system`**; now lists `enum.from()` as a throw site (coherence I-6).
- **docs/milestones.md** — Milestone 2 summary: error code count 31 → **38** (coherence I-1).
- **docs/milestones.md** — Milestone 2 scope: corrected error-code groupings — ref rules E020–E022, named/optional E023–E026, entry point E027–E029; added missing categories (inheritance modifiers E032–E036, reserved keywords E030, arrays E031/E038) (coherence I-2).
- **docs/milestones.md** — Milestone 7: `toUpper`/`toLower` → **`toUpperCase`**/**`toLowerCase`** (coherence VII-5).
- **review/archives/coherence_closed_20260303.md** — Wrong year: 2025 → **2026** (coherence VII-1).
- **README.md** — Project structure: `nlvm/` → **`nlvm-specs/`** (coherence VII-2).

### Added

- **docs/specs.md** — § Switch/Match: documented **fall-through** semantics (without `break`, execution continues into the next case body), previously defined only in vm.md (coherence I-8).
- **docs/stdlib.md** — § Core interfaces (built-in): new section cross-referencing **Stringable**, **Cloneable**, and **ValueEquatable** interfaces from specs.md (coherence V-7).
- **docs/coherence.md** — 19 items resolved (I-1, I-2, I-4, I-6, I-7, I-8, III-1–III-6, III-8, III-11, V-7, VII-1–VII-6).

## 0.8.2 — 2026-03-03

### Added

- **docs/coherence.md** — New coherence tracker: 66 items across 7 categories (cross-document inconsistencies, language spec omissions, incorrect examples, VM/compiler gaps, stdlib issues, under-specified semantics, editorial errors). Full re-audit of all specification documents.

## 0.8.1 — 2026-03-03

### Changed

- **review/archives/coherence_closed_20260303.md** — Coherence tracker moved from `docs/coherence.md` to `archives/` and renamed. All items were resolved; document archived for reference.

## 0.8.0 — 2026-03-02

### Added

- **docs/specs.md** — § Multidimensional arrays: `T[][]` as array of arrays, `new T[n₁][n₂]` fixed-size creation, partial dimensions (`new T[n][]`), initializer lists, chained indexing. Contiguous-suffix rule for omitted dimensions (coherence § V-4).
- **docs/compiler.md** — § Multidimensional array creation: desugaring into nested `NEW_ARRAY` + loop + `ARRAY_STORE`. E038 — non-first dimension size omitted in middle position.

### Changed

- **docs/vm.md** — § Array layout: clarified that multidimensional arrays are nested arrays with `element_type` tag `5` (reference) and `TYPE_DESC` for inner type. § Array operations: note that no new opcode is needed; compilation uses existing `NEW_ARRAY`/`ARRAY_STORE`.
- **docs/compiler.md** — Error code summary: added E038 (Arrays).
- **README.md** — Language highlights: arrays now mention multidimensional (`T[][]`); error code count 37 → 38.
- **docs/coherence.md** — V-4 resolved: multidimensional array creation fully specified.
- **docs/milestones.md** — Error test range E001–E037 → E001–E038.

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
