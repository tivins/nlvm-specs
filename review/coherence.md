# NL Specification — Coherence Tracker

This document lists all known inconsistencies, errors, omissions, and gaps across the NL specification
documents (`specs.md`, `stdlib.md`, `compiler.md`, `vm.md`, `optimizations.md`, `milestones.md`, `tests.md`, `README.md`).
Each item has a checkbox to track resolution.

Audit performed on **2026-03-03**, against spec version **0.8.1**.

---

## Summary

| Category | Count | Description |
|----------|-------|-------------|
| [I. Cross-document inconsistencies](#i-cross-document-inconsistencies) | 8 (6 resolved) | Contradictions between documents |
| [II. Language specification omissions](#ii-language-specification-omissions) | 17 (10 resolved) | Features referenced or implied but never formally defined in specs.md |
| [III. Incorrect examples](#iii-incorrect-examples) | 11 (8 resolved) | Code that would not compile, produces wrong results, or contradicts the spec |
| [IV. VM and compiler specification gaps](#iv-vm-and-compiler-specification-gaps) | 7 (2 resolved) | Missing pieces in vm.md or compiler.md |
| [V. Standard library issues](#v-standard-library-issues) | 8 (4 resolved) | stdlib.md problems (inconsistencies, missing API) |
| [VI. Under-specified semantics](#vi-under-specified-semantics) | 9 (1 resolved) | Defined but incomplete — a compiler/VM implementor cannot proceed without guessing |
| [VII. Documentation and editorial errors](#vii-documentation-and-editorial-errors) | 6 (6 resolved) | Typos, wrong numbers, stale cross-references |
| [VIII. Security-related specification gaps](#viii-security-related-specification-gaps) | 10 (1 resolved) | Missing security hardening, unsafe APIs, unspecified safety behavior — see [security-audit.md](security-audit.md) |

---

## I. Cross-document inconsistencies

### I-1. milestones.md — error code count out of date

- [x] **milestones.md line 14** — Milestone 2 summary says *"31 codes, 1 warning"*. The actual count is **38 error codes** (E001–E038) and 1 warning (W001), consistent with README and compiler.md. The summary table is stale. *(fixed 0.8.3)*

### I-2. milestones.md — Milestone 2 error-code groupings are wrong

- [x] **milestones.md lines 63–71** — The error codes are assigned to the wrong categories: *(fixed 0.8.3)*

| Category (milestones.md) | Codes listed | Correct codes (compiler.md) |
|---|---|---|
| Ref rules | E020–E024 | E020–E022 |
| Named/optional rules | E025–E028 | E023–E026 |
| Entry point validation | E029, E030, E031, E038 | E027–E029 |

E030 (reserved keywords), E031 (arrays), E032–E036 (abstract/final), E037 (template bound), E038 (multidimensional arrays) are not listed or are assigned to the wrong section.

### I-3. For-each loop — `const` required or optional?

- [x] **specs.md** only showed the form `for (const auto item : collection)`. **stdlib.md** examples omitted `const`. *(fixed 0.8.4)*
  - specs.md § Loops: `const` is **optional**; both `for (auto item : collection)` and `for (const auto item : collection)` are valid.
  - Added **implicit const in const context**: in a const method, when iterating over `this.property`, or when iterating over a const parameter, the loop variable is implicitly read-only (E039).

### I-4. `system.Env` listed as a namespace

- [x] **stdlib.md line 3** lists `system.Env` as a **namespace**: *"All types live in the `system`, `system.io`, `system.net`, `system.thread`, `system.time`, **`system.Env`**, `system.ps`, and `system.text` namespaces…"*. But `system.Env` is a **class** (with static methods) inside the `system` namespace. It should be removed from the namespace list. *(fixed 0.8.3)*

### I-5. `throws` declared for runtime exceptions in stdlib

- [x] **stdlib.md** — `system.Int.parse` and `system.Float.parse` are declared `throws NumberFormatException`. But `NumberFormatException extends RuntimeException`, and the spec states (compiler.md § Checked exception propagation): *"Runtime exceptions (`RuntimeException` and subclasses) are exempt: they do not require `throws` declarations."* *(fixed 0.8.24: option (b) — compiler.md § Checked exception propagation now states that `throws` may list runtime exceptions for documentation purposes; the compiler does not enforce them)*

### I-6. `IllegalArgumentException` namespace attribution

- [x] **stdlib.md exceptions table (line 897)** lists `IllegalArgumentException` under namespace `system.time` because it is thrown by `TimeZone.get()`. But `IllegalArgumentException` is also thrown by `enum.from()` (specs.md § Enums methods), which is a core language feature — not in `system.time`. The exception belongs in the `system` namespace (alongside `IndexOutOfBoundsException`, `NumberFormatException`, etc.). The table should be corrected. *(fixed 0.8.3)*

### I-7. `system.Out.print` — "may" provide overloads, but examples require them

- [x] **stdlib.md line 154** says *"Overloads of `print` and `println` for other types (e.g. `int`, `float`, `bool`) **may** be provided by the runtime."* But **specs.md** examples call `system.Out.print(counter)` (line 1552, `counter` is `int`), `system.Out.print(obj.value)` (line 1018, `int`), etc. These would be compile errors without overloads (cannot pass `int` to a `string` parameter). The "may" must be changed to **"must"**, or the examples must use explicit `(string)` casts / concatenation. *(fixed 0.8.3: "may" → "must")*

### I-8. `switch` fall-through — defined in vm.md only

- [x] **vm.md § Switch statements** states: *"Without `break`, execution falls through to the next case body."* But **specs.md** never defines fall-through semantics for `switch`. The language-level behavior should be defined in specs.md (the language spec), not only in the compilation strategy. *(fixed 0.8.3: added fall-through semantics in specs.md § Switch/Match)*

---

## II. Language specification omissions

### II-1. No `instanceof` keyword or expression

- [ ] **vm.md** defines an `INSTANCEOF` opcode (line 472). **milestones.md** (line 38) lists `instanceof` as an expression. But **specs.md** does not define an `instanceof` keyword, does not list it in the keywords table, and does not describe any expression syntax for runtime type testing. A compiler implementor cannot generate `INSTANCEOF` without knowing the source-level syntax. Suggested: add `instanceof` as a binary expression `expr instanceof ClassName` → `bool`.

### II-2. No complete operator precedence table

- [ ] **specs.md § Operator precedence and usage** gives only partial relative precedence for `??`, `?:`, and `? :`. A full operator precedence table (from highest to lowest) is missing. Without it, expressions mixing multiple operators have ambiguous parsing.

### II-3. Variable shadowing rules not specified

- [x] Can a local variable shadow a class field? Can a block-scoped variable shadow an enclosing block's variable? Specs.md does not define shadowing rules. *(fixed 0.8.21: specs.md § Variable shadowing — both forms allowed; local/parameter shadows field, inner block shadows outer; compiler.md cross-reference added)*

### II-4. Byte literal syntax not defined

- [x] The spec defines `byte` as a type but provides no syntax for byte literals. The only way to obtain a byte is through `(byte) intExpr`. If byte literals are not supported, this should be stated explicitly. *(fixed 0.8.19: specs.md § Native types — explicitly documents that byte literals are not supported; use `(byte) intExpr`)*

### II-5. Float literal format not specified

- [x] **specs.md** uses `.1`, `.2`, `.3` as float literals (line 1145). But the spec never defines the accepted float literal formats. *(fixed 0.8.24: specs.md § Float literal format — formats `3.14`, `.5`, `2.`, `0.0` defined; scientific notation not supported)*

### II-6. `const` on local variables

- [x] The `const` keyword is used for method declarations and parameters. For-each loops use `const auto`. But can a local variable be declared `const`? E.g. `const int x = 42;`. The spec neither allows nor disallows this. *(fixed 0.8.20: specs.md § Const methods and parameters — local variables may be declared `const`; they cannot be reassigned after initial assignment)*

### II-7. Multiple interface implementation syntax

- [x] **vm.md module format** has `interfaces_count` and `interfaces[]`, supporting multiple interfaces. But **specs.md** only showed single-interface syntax. *(fixed 0.8.24: specs.md § Extends, Implements — comma-separated syntax `implements A, B, C` defined with example)*

### II-8. Interface extending interface

- [ ] Can interfaces extend other interfaces? E.g. `interface Closeable extends Disposable`. The spec shows classes extending classes and classes implementing interfaces, but never interface-to-interface inheritance.

### II-9. Constructor chaining (`this(…)`)

- [ ] In Java, a constructor can call another overloaded constructor with `this(args)`. The NL spec only defines `super(args)` for parent constructor calls. Constructor chaining within the same class is not mentioned.

### II-10. Enum custom methods and properties

- [x] Can enums have custom methods and properties beyond the built-in `from()`, `tryFrom()`, and `value`? The spec doesn't say. Many languages allow enum methods; NL's position is unstated. *(fixed 0.8.27: specs.md § Custom methods and properties — enums may declare custom static/instance methods and static properties; style recommendation to keep enums lightweight)*

### II-11. `match` expression — `default` clause mandatory?

- [ ] If no arm of a `match` expression matches and there is no `default`, what happens? Is `default` mandatory? Unreachable `default` for exhaustive matches? The spec is silent.

### II-12. `this` in static context

- [x] The spec should explicitly state that `this`, `Self`, and instance member access are forbidden in `static` methods. *(fixed 0.8.24: specs.md § Static methods + compiler.md § Static context restrictions — E040 added)*

### II-13. Nested / inner classes

- [x] The one-class-per-file rule (specs.md § Source code files) implies no nested classes, but this was not explicitly stated. *(fixed 0.8.24: specs.md § Source code files — explicitly states nested class definitions are not allowed)*

### II-14. For-loop — multiple init variables

- [x] Can the init part of `for (init; cond; incr)` declare multiple variables? *(fixed 0.8.24: specs.md § Loops — multiple same-type declarations allowed, e.g. `for (int i = 0, j = 10; …; …)`)*

### II-15. Scoping of for-loop init variable

- [x] Is the variable declared in `for (int i = 0; …; …)` scoped to the for block or visible after it? *(fixed 0.8.24: specs.md § Loops — variables declared in for-loop init are scoped to the for block)*

### II-16. `Self` in interface context

- [ ] `Cloneable` defines `public Self clone()`. In the interface, `Self` refers to "the implementing class" — a covariant return type. This semantic is used but never formally defined. What does `Self` resolve to inside an interface body? How does the compiler handle it?

### II-17. Duplicate method signatures

- [x] What happens if a class declares two methods with the same name and parameter types (exact duplicates)? *(fixed 0.8.24: compiler.md § Duplicate definitions — E041 for duplicate method, E042 for duplicate class; also resolves IV-6)*

---

## III. Incorrect examples

### III-1. system.String example — wrong `length()` result

- [x] **stdlib.md line 288** — `int n = text.length(); // 15`. The string `"  Hello, World  "` has **16** characters (2 spaces + `Hello, World` + 2 spaces), not 15. *(fixed 0.8.3)*

### III-2. system.String example — wrong `substring()` result

- [x] **stdlib.md line 294** — `string sub = text.substring(2, 8); // "Hello"`. With 0-based indexing, `substring(2, 8)` returns characters at indices 2–7 = `"Hello,"` (6 characters, including the comma), not `"Hello"` (5 characters). *(fixed 0.8.3)*

### III-3. Template class example — duplicate variable name

- [x] **specs.md lines 1144–1145:**
  ```
  Vector<int> v1 = new Vector<int>(1, 2, 3);
  Vector<float> v1 = new Vector<float>(.1, .2, .3);
  ```
  Both variables are named `v1`. The second should be `v2`. *(fixed 0.8.3)*

### III-4. Nullish coalescing example — non-nullable assigned null

- [x] **specs.md lines 1857–1858:**
  ```
  MyObject a = null;
  system.Out.print(a ?? "is null");
  ```
  `MyObject` without `|null` is non-nullable. Assigning `null` to it is compile error **E003**. Should be `MyObject|null a = null;`. *(fixed 0.8.3: changed to `string|null` for type consistency)*

### III-5. Nullish coalescing example — non-nullable with `??`

- [x] **specs.md lines 1861–1862:**
  ```
  MyObject b = getObject();
  system.Out.print(b ?? "default");
  ```
  If `b` is `MyObject` (non-nullable), it can never be `null`, so `??` is vacuous. The type should be `MyObject|null`. *(fixed 0.8.3: changed to `string|null` for type consistency)*

### III-6. Elvis operator examples — non-nullable and type mismatch

- [x] **specs.md line 1886–1887:**
  ```
  MyObject obj = null;
  string name = obj ?: "default";
  ```
  Same E003 issue (`MyObject obj = null`). Additionally, the result type of `obj ?: "default"` is unclear when `obj` is `MyObject` and the default is `string` — these are unrelated types. *(fixed 0.8.3: changed to `string|null` for type consistency)*

### III-7. Elvis operator examples — type incompatibility

- [x] **specs.md** — Elvis operator examples used incompatible types (`int ?: string`, `bool ?: string`). *(fixed 0.8.24: examples rewritten with same-type operands; type compatibility note added)*

### III-8. Fluent method `save()` — missing return statement

- [x] **specs.md lines 981–984:**
  ```
  public Self save() {
      // save in database, or whatever.
  }
  ```
  The method declares return type `Self` but has no `return this;` statement. While clearly a placeholder, it looks like valid code that would fail compilation (missing return value). *(fixed 0.8.3: added `return this;`)*

### III-9. Grep example — missing `const` in for-each

- [x] **stdlib.md line 567:** `for (auto m : matches)` — `const` is optional in for-each (specs.md § Loops); the example is valid. *(resolved with I-3)*

### III-10. Process example — missing `const` in for-each

- [x] **stdlib.md line 842:** `for (auto p : processes)` — same as III-9; valid. *(resolved with I-3)*

### III-11. `system.io.Path.join` example — wrong array syntax

- [x] **stdlib.md line 531:**
  ```
  string path = system.io.Path.join(new string[] {"src", "com", "example", "App.nl"});
  ```
  Per specs.md § Arrays, array initializer syntax is `new T[]{ ... }` (no space between `[]` and `{`). The example uses `new string[] {"src", ...}` with a space. This is ambiguous — either the syntax allows a space (not specified) or the example is wrong. *(fixed 0.8.3: removed space)*

---

## IV. VM and compiler specification gaps

### IV-1. Class and method flags missing `ABSTRACT` and `FINAL`

- [ ] **vm.md § Class descriptor (line 150–155)** defines class flag bits 0 (`READONLY`), 1 (`INTERFACE`), 2 (`ENUM`). There are no bits for `ABSTRACT` or `FINAL` classes.

  **vm.md § Method descriptor (line 233–243)** defines method flag bits 0–7 but lacks `ABSTRACT` and `FINAL` method flags.

  These flags are required to enforce abstract/final semantics at the VM level (e.g., rejecting instantiation of abstract classes, preventing override of final methods). Two new class bits and two new method bits are needed.

### IV-2. Module format — no debug / line-number table

- [ ] **vm.md § Stack trace construction** references *"a line-number table embedded in the method's debug info"*. But the module format (§ Method descriptor) has no field for a line-number table or debug info section. An implementor cannot produce stack traces with line numbers without this structure.

### IV-3. Abstract method representation in module format

- [ ] Abstract methods have no body. The module format always emits `code_length` + `code[]` for methods. The spec should state that abstract methods have `code_length = 0` and an empty `code` array (or that the method descriptor uses the `ABSTRACT` flag to indicate no code).

### IV-4. Conditional branch width limitation

- [ ] `IF_TRUE` and `IF_FALSE` use `i16` offset (max ±32 KiB). `GOTO_W` exists with `i32` offset for wide jumps. But there are no `IF_TRUE_W` / `IF_FALSE_W` equivalents. For methods with code > 32 KiB, conditional branches cannot reach distant targets. The workaround (invert condition + GOTO_W) should be documented or wide conditional branch opcodes should be added.

### IV-5. `F2I` clamping — inconsistent with specs.md

- [x] **vm.md F2I** says *"Values outside `int` range are clamped to INT_MIN / INT_MAX."* But **specs.md § Type conversions** said *"undefined behavior."* *(fixed 0.8.24: specs.md aligned to vm.md — clamping to INT_MIN/INT_MAX, safety-first)*

### IV-6. No error code for duplicate method/class definitions

- [x] If the same fully qualified class name appears in two modules, or if a class defines two methods with identical signatures, there was no error code. *(fixed 0.8.24: compiler.md § Duplicate definitions — E041 and E042 added; see also II-17)*

### IV-7. Exception `readonly` vs VM `stackTrace` assignment

- [ ] The exception hierarchy declares `class readonly Exception` with field `public ExecutionPoint[] stackTrace`. In a readonly class, fields can only be assigned inside `construct`. But the stack trace is populated by the VM when the exception object is created — **after** the constructor runs. This requires the VM to bypass readonly enforcement for this specific field, which contradicts the spec. Either `stackTrace` should not be a regular field (use a native/internal mechanism), or the readonly rule needs an exception for VM-populated fields.

---

## V. Standard library issues

### V-1. `system.Bool` — no `parse` method

- [x] `system.Int` has `parse`, `system.Float` has `parse`, but `system.Bool` had only `toString`. There was no `parse(string s)` for converting `"true"` / `"false"` to `bool`. This was an API asymmetry. *(fixed 0.8.28: stdlib.md § system.Bool — added parse; 0.8.29: renamed parseInt/parseFloat/parseBool to parse across Int, Float, Bool)*

### V-2. `system.String` — `trim` and `split` were static but peers are instance methods

- [x] String instance methods (`length`, `charAt`, `substring`, `indexOf`, `contains`, `replace`, `toUpperCase`, `toLowerCase`, `startsWith`, `endsWith`) are called on string values: `text.length()`. But `trim` and `split` were static methods: `system.String.trim(text)` and `system.String.split(s, delim)`. This asymmetry forced users to remember which methods are where. *(fixed 0.8.30: stdlib.md § system.String — trim() and split(delimiter) are now instance methods; static forms trim(s) and split(s, delim) kept for flexibility)*

### V-3. `system.List<T>` — no positional `remove` or `contains`

- [ ] `system.List<T>` provides `pushBack`, `pushFront`, `popBack`, `popFront`, `get`, `set`, `add`, `size`, and for-each support. But there is no `remove(int index)` for arbitrary-position removal and no `contains(T value)` for searching. These are common operations on dynamic lists.

### V-4. `system.thread.Thread` — no `isAlive()` method

- [ ] There is no way to check if a thread is still running without blocking on `join()`. An `isAlive()` method (returning `bool`) is standard in most threading APIs.

### V-5. Thread safety of collections not documented

- [ ] `system.List<T>` and `system.Map<K, V>` are heap objects shared across threads (vm.md § Threading model). The spec does not state whether they are thread-safe or require external synchronization. This must be documented (most likely: *not* thread-safe — caller must use a `Mutex`).

### V-6. `system.io.File.open` — no file mode parameter

- [ ] `File.open(string path)` opens a file for "reading and/or writing." There is no parameter to control the open mode (read-only, write-only, append, truncate, create). This makes the API unsuitable for many use cases.

### V-7. `Stringable`, `Cloneable`, `ValueEquatable` not listed in stdlib

- [x] These three core interfaces are defined in **specs.md** (§ Extends/Implements, § Cloneable interface, § ValueEquatable interface) but never mentioned in **stdlib.md**. They are part of the language/runtime contract and should appear in stdlib.md (or at least be cross-referenced) since implementors need to know they are built-in. *(fixed 0.8.3: added § Core interfaces (built-in) in stdlib.md with cross-references)*

### V-8. No `StackOverflowException` in exception hierarchy

- [x] Infinite recursion will exhaust the call stack. There was no `StackOverflowException` in the exception hierarchy. *(fixed 0.8.24: added `StackOverflowException extends RuntimeException` in specs.md § Exception class hierarchy, stdlib.md exceptions table, and vm.md § Call frame)*

---

## VI. Under-specified semantics

### VI-1. Ternary operator — result type rules

- [ ] `condition ? a : b` requires *"compatible types"* for both branches, but "compatible" is not defined. Can `int` and `float` be branches (with implicit widening)? Can `null` and `T` be branches (producing `T|null`)? Can a subclass and a superclass be branches? A table of type-compatibility rules for the ternary operator is needed.

### VI-2. Elvis and nullish coalescing — result type rules

- [ ] `a ?? b` and `a ?: b` — when `a` and `b` have different types, what is the result type? The examples show `int ?: string` and `MyObject ?? string`, but the resulting type is never defined. For a statically-typed language, this is critical.

### VI-3. Union type narrowing (smart casts)

- [ ] **compiler.md § Union type compatibility** says narrowing requires *"an `if` check, `match`, or null comparison"*, but the exact narrowing rules are not defined. After `if (x != null)`, is `x` narrowed from `T|null` to `T` within the `if` body? What about `if (x instanceof SomeClass)`? These flow-sensitive typing rules must be specified for a compiler to implement them.

### VI-4. Anonymous function return type deduction

- [ ] Specs.md says the return type of anonymous functions *"can be explicitly specified or automatically deduced."* But the deduction rules are not specified. What if a lambda has multiple return paths with different types? What if the body is a single expression — is the return type deduced from it?

### VI-5. Operator overloading — which operators, exact rules

- [ ] Specs.md says *"operators such as `+`, `-`, `*`, `/`, `==`, etc."* can be overloaded. The exhaustive list of overloadable operators is missing. Can `==` be overloaded alongside `ValueEquatable`? Can `[]` (subscript) be overloaded? Can unary `!`, `-`, `++`, `--`? What are the required signatures for each?

### VI-6. `match` expression — exhaustiveness

- [ ] For enums, is `match` required to be exhaustive (cover all cases)? For integers/strings, since the domain is infinite, is `default` always required? This affects compiler analysis.

### VI-7. Multiple `catch` blocks for same exception type

- [ ] Can two `catch` blocks in the same try/catch catch the same exception type? If so, which one wins? If not, there should be an error code.

### VI-8. `readonly` class interaction with `abstract` and `final`

- [ ] Can a class be `abstract class readonly`? `final class readonly`? The exception hierarchy shows `class readonly Exception` (not abstract, not final), and `readonly` is used on many exception subclasses. But the interaction rules are not stated. The modifier ordering (`class readonly`, `abstract class readonly`, `final class readonly`) is also not formalized.

### VI-9. `system.ps.Process.exit` — control flow impact

- [x] `Process.exit(int code)` is documented as *"Does not return."* But the compiler didn't know the code after it is unreachable. *(fixed 0.8.24: compiler.md § Terminal statements — `return`, `throw`, `Process.exit()` defined as terminal; stdlib.md updated)*

---

## VII. Documentation and editorial errors

### VII-1. Archived coherence document — wrong year

- [x] **review/archives/coherence_closed_20260303.md line 10** says *"All items resolved as of **2025**-03-03."* Should be **2026**-03-03. *(fixed 0.8.3)*

### VII-2. README project structure — folder name

- [x] **README.md line 25** shows the project root as `nlvm/`. The actual repository is named `nlvm-specs`. This is cosmetic but potentially confusing. *(fixed 0.8.3)*

### VII-3. specs.md line 1404 — "tailing" → "trailing"

- [x] *"final tailing coma is valid"* — should be **"trailing comma"**. *(fixed 0.8.3)*

### VII-4. specs.md line 1197 — ternary in `max` with unchecked comparisons

- [x] `return a > b ? a : b;` — the ternary is used on template type `T`. The `>` operator may not be defined for `T` if no bound is specified. The compiler would catch this (E006 at instantiation), but the example is misleading since it appears in a section about template methods with no bound. A comment or bounded example would be clearer. *(accepted 0.8.3: this is by design — unbounded templates are checked at instantiation per § Template instantiation; the example demonstrates this pattern intentionally)*

### VII-5. milestones.md § Milestone 7 — `toUpper`/`toLower` vs `toUpperCase`/`toLowerCase`

- [x] **milestones.md line 201** lists *"`toUpper`, `toLower`"*. The actual method names in stdlib.md are **`toUpperCase`** and **`toLowerCase`**. The milestone reference is wrong. *(fixed 0.8.3)*

### VII-6. specs.md — no link reference for `[23]` (`ref`)

- [x] Keywords table uses `[23]` for `ref` → `#parameter-passing-semantics`. The anchor link works but the numerical reference style is inconsistent with the labels used elsewhere. (Minor — formatting only.) *(accepted 0.8.3: all keyword links use numbered references consistently; this is the intended style)*

---

## VIII. Security-related specification gaps

*See [review/security-audit.md](security-audit.md) for the full security audit (26 findings). The items below track specification changes needed to address the most impactful findings.*

### VIII-1. `system.ps.Process.run(string)` — no command injection warning

- [ ] **stdlib.md § system.ps.Process** — `Process.run(string command)` passes input to the platform shell. The spec does not warn about command injection or recommend the `run(string[] args)` overload as the safe alternative. *[SEC-01]*

### VIII-2. File system APIs — no path traversal warning

- [ ] **stdlib.md § system.io** — `File.open`, `File.readAllText`, `File.writeAllText`, `File.glob`, `Directory.create`, `Directory.remove`, `Process.setCwd` perform no path sanitization. The spec does not document that path validation is the caller's responsibility. *[SEC-02]*

### VIII-3. No `tryParseInt` / `tryParseFloat` safe parsing methods

- [ ] **stdlib.md § system.Int, system.Float** — `parse` throws `NumberFormatException` (a `RuntimeException`), meaning callers are not required to handle parse errors. No safe-by-default alternative (`tryParse(string) : int|null`) exists. *[SEC-05]*

### VIII-4. Integer overflow behavior undocumented in specs.md

- [ ] **specs.md § Native types** — vm.md specifies that `int` arithmetic wraps on overflow (two's complement), but specs.md does not mention overflow behavior. Developers may not be aware of wrapping semantics. The comparator example `(int a, int b) => a - b` (specs.md line 304) is vulnerable to overflow. *[SEC-10]*

### VIII-5. Byte array I/O bounds checking unspecified

- [ ] **stdlib.md § system.io.FileHandle, system.net.TcpStream, system.net.UdpSocket** — `read(byte[] buffer, int offset, int length)` and `write(byte[] data, int offset, int length)` do not specify behavior when `offset` or `length` are negative, or when `offset + length > buffer.length()`. In native implementations, this could cause buffer over-reads or over-writes. *[SEC-16]*

### VIII-6. Network APIs — no TLS certificate validation requirement

- [ ] **stdlib.md § system.net.Http** — `Http.get`/`Http.post` accept `https://` URLs but the spec does not require certificate validation. An implementation that skips validation would be vulnerable to MITM attacks. *[SEC-08]*

### VIII-7. No cryptographically secure random number generator

- [ ] **stdlib.md § system.Random, system.Uuid** — `system.Random` is a PRNG, unsuitable for security purposes. `system.Uuid.random()` does not specify UUID version or randomness source. No `system.SecureRandom` (CSPRNG) exists. *[SEC-09]*

### VIII-8. Read/write after close behavior unspecified

- [x] **stdlib.md § system.io.FileHandle, system.net.TcpStream, system.net.UdpSocket** — `close()` is idempotent, but the behavior of `read()`, `write()`, `readLine()`, `flush()` after `close()` is not specified. Should throw `IOException`. *[SEC-11]* *(fixed 0.8.23)*

### VIII-9. `Regex.match` — full vs partial match unspecified

- [ ] **stdlib.md § system.text.Regex** — `Regex.match(string pattern, string input)` returns `bool`, but it is not specified whether this is a **full match** (the entire input must match the pattern) or a **partial match** (the pattern must be found somewhere in the input). No `Regex.escape()` utility exists for safely using user input in patterns. *[SEC-23]*

### VIII-10. Module format — no integrity verification

- [ ] **vm.md § Module format** — The `.nlm` format has a 4-byte magic number but no hash, checksum, or digital signature. A modified module can be loaded without detection. *[SEC-03]*

---

## Previously resolved

See [archives/coherence_closed_20260303.md](archives/coherence_closed_20260303.md) for the first coherence pass (20 items, all resolved in versions 0.3.1–0.8.1).
