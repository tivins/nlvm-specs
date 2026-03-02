# NL Specification — Coherence Tracker

This document lists all known inconsistencies, errors, and gaps across the NL specification
documents (`specs.md`, `stdlib.md`, `compiler.md`, `vm.md`). Each item has a checkbox to track
resolution. When an item is fixed, check the box and note the date or version.

Sources: analysis from `docs_internal/coherence_2.md` and `docs_internal/documentation_of_virtual_machine.md`,
plus a full re-audit of `docs/` performed on 2026-03-01.

---

## Summary

| Category | Count | Description |
|----------|-------|-------------|
| [I. Syntax violations](#i-syntax-violations) | 3 | Direct violations of spec rules in spec text itself |
| [II. Nonexistent references](#ii-nonexistent-references) | 3 | Methods, types, or functions used but never defined |
| [III. Cross-document contradictions](#iii-cross-document-contradictions) | 2 | specs.md vs compiler.md, or stdlib.md vs specs.md |
| [IV. Incorrect or misleading examples](#iv-incorrect-or-misleading-examples) | 4 | Code examples that would not compile or behave as shown |
| [V. Under-specified elements](#v-under-specified-elements) | 1 | Features used or referenced but incompletely defined |
| [VI. Missing features](#vi-missing-features) | 4 | Identified gaps (not blocking but tracked) |

---

## I. Syntax violations

The spec explicitly states: *"The suffix `?` is not accepted; the language requires the explicit union with `null`."*
([specs.md § Union types](specs.md#union-types-and-explicit-nullable)). Yet `?` is used in several places.

### I-1. `Self?` in enum tryFrom signature

- [x] **specs.md:1217** — `static Self? tryFrom(string value)` → `static Self|null tryFrom(string value)` *(fixed 2026-03-01)*

### I-2. `string?` in FileHandle.readLine

- [x] **stdlib.md:438** — `string? readLine() throws IOException` → `string|null readLine() throws IOException` *(fixed 2026-03-01)*
- [x] **stdlib.md:450** — `string? line;` in the FileHandle example → `string|null line;` *(fixed 2026-03-01)*

### I-3. `string[]?` in HttpResponse.headers

- [x] **stdlib.md:104** — `string[]` followed by `?` in the type column of `HttpResponse.headers`. Should be
  `string[]|null` if nullable, or documented differently if the field itself is optional. *(fixed 2026-03-01)*

---

## II. Nonexistent references

### II-1. `system.Out.writeLine` does not exist

- [x] **stdlib.md:452** — `system.Out.writeLine(line)` in the FileHandle example. `system.Out` only defines
  `print` and `println` (stdlib.md:143-144). Should be `system.Out.println(line)`. *(fixed 2026-03-01)*

### II-2. `char` type referenced but never defined

- [x] **specs.md:729** — "Scalar types (`int`, `float`, `bool`, `char`, `byte`, etc.)" in the
  Parameter passing semantics section. `char` is not listed in native types (specs.md:199) and has no
  definition anywhere. Remove `char` from the list or define it as a native type.
  *(fixed 2026-03-02: removed `char` from scalar list; documented that character = string length 1; added Planned section for future `char` type.)*

### II-3. `assert()` used but never defined

- [x] **specs.md:1227-1233** — `assert(status == Status.NotFound)` and similar calls in the enum example.
  `assert` is not a keyword, not in the stdlib, and not defined anywhere. Either:
  - Add `assert` as a built-in statement/keyword, or
  - Replace the example with explicit `if` checks or remove it.
  *(fixed 2026-03-01: replaced with explicit `if` checks.)*

---

## III. Cross-document contradictions

### III-1. Exception inheritance rules — specs.md vs compiler.md

This is the most significant inconsistency. The two documents define **different rules**.

**specs.md:2017** says:
> "the overriding method must declare at least the same checked exceptions (or their subclasses).
> It cannot declare fewer checked exceptions than the base method."

**compiler.md:235-242** says:
> "For each exception type E in the parent's throws clause, the child's throws clause must include
> E **or a subclass of E**."
> "The child may not introduce **new** checked exception types that are not subtypes of exceptions
> declared by the parent."

These are different rules. Consider: parent declares `throws Exception, IOException`.

| Child declaration | specs.md | compiler.md (Liskov) |
|-------------------|----------|----------------------|
| `throws IOException` | **Error** (fewer exceptions) | **Valid** (IOException covers both Exception and IOException since it extends Exception) |
| `throws FileNotFoundException, IOException` | OK (example at specs.md:2033) | Valid |

Additionally, specs.md contradicts **itself**:
- Line 2033 marks `throws FileNotFoundException, IOException` as OK (replacing `Exception` with a sub-sub-class).
- Line 2038-2041 marks `throws IOException` as Error ("Missing Exception"). But by the same logic,
  `IOException` is a subclass of `Exception` and should be valid.

- [x] **Resolved (2026-03-02):** adopted the compiler.md rule (Liskov-compatible, matches Java). Rewrote
  specs.md § Exception inheritance rules and examples to match. E016/E017 descriptions in compiler.md
  were already correct.

### III-2. Array built-in methods summary incomplete in stdlib.md

- [x] **Resolved (2026-03-02):** stdlib.md § Arrays summary now includes **`forEach()`**, **`sort()`**, **`find()`**
  to match specs.md and vm.md.

---

## IV. Incorrect or misleading examples

### IV-1. `MyException` constructor missing `super(message)`

- [x] **Resolved (2026-03-02):** Added `super(message);` as first statement in MyException constructor.

### IV-2. Entry point example — wrong argc check

- [x] **Resolved (2026-03-02):** Changed `argc < 3` to `argc < 4` in Calculator example.

### IV-3. Non-nullable parameter compared to null

- [x] **Resolved (2026-03-02):** Anonymous function examples: changed parameter types from `string` to
  `string|null` where the code compares to null (`input` and `s`), so the null check is meaningful.

### IV-4. Enum descriptions in French

- [x] **Resolved (2026-03-02):** Translated `from()` / `tryFrom()` descriptions to English.

---

## V. Under-specified elements

### V-1. Exception type for `enum.from()` not specified

- [x] **specs.md:1220** — Says "lève une exception" (throws an exception) but does not specify which
  exception class. Should be a specific `RuntimeException` subclass (e.g. `IllegalArgumentException`)
  so that user code and the VM know what to catch.
  *(fixed 2026-03-01: specified `IllegalArgumentException`.)*

### V-2. Keywords declared but never defined

- [x] **Resolved (2026-03-02):** All five items specified in specs.md. **`virtual`** — Removed from keywords; all instance methods are virtual by default (Java-style), documented in § Virtual method dispatch. **`abstract`** — § Abstract classes and methods: abstract class, abstract method, rules. **`final`** — § Final classes and methods: final class (no extends), final method (no override). **`clone`** — Removed from keywords; replaced by Cloneable interface with `clone()` method (§ Cloneable interface); shallow copy by default. **`delete`** — Removed from keywords; GC-only, no manual destruction; destructor wording corrected.

### V-3. Map iteration not defined

- [x] **Resolved (2026-03-02):** Added concrete iteration API to `system.Map<K,V>`: **`keys()`** returning `K[]`,
  **`values()`** returning `V[]`, **`entries()`** returning `MapEntry<K,V>[]`, and **`forEach((K key, V value) => void f)`**.
  Added **`system.MapEntry<K,V>`** result type (stdlib.md § Result types). Maps support the **for-each loop**
  (`for (const auto entry : map)`). Iteration order is consistent across methods but implementation-defined.
  VM desugaring specified in vm.md § For-each loops. Native binding in vm.md § Standard library binding.

### V-4. Multidimensional array creation not specified

- [ ] **specs.md:486,495** — `typedef int[][] Matrix` and `new int[size][size]` are used in the typedef example,
  but the spec only defines `new T[n]` (1D fixed size) and `new T[]{ ... }` (1D initializer list).
  The syntax `new int[size][size]` for 2D arrays is never defined. Either:
  - Define multidimensional array creation syntax, or
  - Replace the example with nested 1D arrays.

### V-5. Naming convention inconsistency — lowercase modules

- [x] **stdlib.md** — `system.env`, `system.ps`, `system.process` use lowercase names while all other
  stdlib entities use PascalCase (`system.Out`, `system.Int`, `system.io.File`, etc.). Decision needed:
  - If these are **namespaces** containing static functions (not classes), document this distinction.
  - If these are **classes**, rename to `system.Env`, `system.Ps`, `system.Process`.
  *(fixed 2026-03-01: no static functions in namespaces; `system.env` → `system.Env` (class); `system.ps` and `system.process` merged into namespace `system.ps` with class `system.ps.Process` (static list, run, pid, getCwd, setCwd, exit), result types `ProcessInfo` and `ProcessResult`.)*

---

## VI. Missing features (tracked, not blocking)

These are identified gaps — not bugs or inconsistencies, but features whose absence has been noted. They are
tracked here for completeness. Some may be intentional design choices.

### VI-1. `do-while` loop

- [x] **Won't fix:** NL will not implement `do-while`. `while` with `break` is sufficient.

### VI-2. try-with-resources / RAII

- [x] **Noted in specs.md § Planned:** No mechanism specified for now; destructor timing remains implementation-defined. A dedicated construct may be considered in a future version.

### VI-3. `equals()` / `hashCode()` convention

- [x] **Resolved (2026-03-02):** Added **ValueEquatable** interface with `valueEquals(const Self|null other)` and `valueHash()` in specs.md § ValueEquatable interface. `==` on references remains identity-based (vm.md). `system.Map<K,V>` uses valueEquals/valueHash when K implements ValueEquatable (stdlib.md, vm.md).

### VI-4. Bounded generics

- [x] **Resolved (2026-03-02):** Added `template <type T extends Bound>` syntax in specs.md § Bounded type parameters.
  Compiler verifies bound at instantiation (compiler.md § Template instantiation, E037). Enables earlier compile-time
  errors and documentation of template contracts.

### VI-5. Multiple return values / tuples

- [x] **Won't fix:** NL will not support tuples or multiple return values. Use a custom class to return
  multiple values.

---

## Previously resolved (for reference)

The following issues from `docs_internal/coherence_2.md` have been verified as resolved:

| Issue | Resolution |
|-------|-----------|
| `self` vs `Self` ambiguity | specs.md § Self and type keywords — `Self` (type alias, mutation), `this` (instance), `type` (new instance) |
| `undefined` status | Removed from type system; reserved keyword only; definite assignment analysis + default values |
| `swap` / parameter passing semantics | specs.md § Parameter passing semantics — value, reference-value, `ref`, `const ref` |
| `enum.from()` missing `static` | Corrected to `static Self from(...)` |
| `FileHandle` with no read/write | stdlib.md § FileHandle — `read`, `readLine`, `write`, `flush` added |
| No string instance methods | stdlib.md § Instance methods on string — `length`, `charAt`, `substring`, `indexOf`, etc. |
| No int/float to string conversion | `system.Int.toString`, `system.Float.toString`, `system.Bool.toString` |
| No `continue` in loops | specs.md § Loops — `continue` documented |
| String concatenation rules | specs.md § String concatenation — conversion rules defined |
| Cast / type conversions | specs.md § Type conversions and casting — full section with rules |
