# NL Specification — Coherence Tracker (Resolved)

Archived items from [review/coherence.md](../coherence.md), resolved as of **2026-03-05**.

---

## I. Cross-document inconsistencies (8 resolved)

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

## II. Language specification omissions (11 resolved)

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

### II-10. Enum custom methods and properties

- [x] Can enums have custom methods and properties beyond the built-in `from()`, `tryFrom()`, and `value`? The spec doesn't say. Many languages allow enum methods; NL's position is unstated. *(fixed 0.8.27: specs.md § Custom methods and properties — enums may declare custom static/instance methods and static properties; style recommendation to keep enums lightweight)*

### II-12. `this` in static context

- [x] The spec should explicitly state that `this`, `Self`, and instance member access are forbidden in `static` methods. *(fixed 0.8.24: specs.md § Static methods + compiler.md § Static context restrictions — E040 added)*

### II-13. Nested / inner classes

- [x] The one-class-per-file rule (specs.md § Source code files) implies no nested classes, but this was not explicitly stated. *(fixed 0.8.24: specs.md § Source code files — explicitly states nested class definitions are not allowed)*

### II-14. For-loop — multiple init variables

- [x] Can the init part of `for (init; cond; incr)` declare multiple variables? *(fixed 0.8.24: specs.md § Loops — multiple same-type declarations allowed, e.g. `for (int i = 0, j = 10; …; …)`)*

### II-15. Scoping of for-loop init variable

- [x] Is the variable declared in `for (int i = 0; …; …)` scoped to the for block or visible after it? *(fixed 0.8.24: specs.md § Loops — variables declared in for-loop init are scoped to the for block)*

### II-17. Duplicate method signatures

- [x] What happens if a class declares two methods with the same name and parameter types (exact duplicates)? *(fixed 0.8.24: compiler.md § Duplicate definitions — E041 for duplicate method, E042 for duplicate class; also resolves IV-6)*

---

## III. Incorrect examples (11 resolved)

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

## IV. VM and compiler specification gaps (3 resolved)

### IV-5. `F2I` clamping — inconsistent with specs.md

- [x] **vm.md F2I** says *"Values outside `int` range are clamped to INT_MIN / INT_MAX."* But **specs.md § Type conversions** said *"undefined behavior."* *(fixed 0.8.24: specs.md aligned to vm.md — clamping to INT_MIN/INT_MAX, safety-first)*

### IV-6. No error code for duplicate method/class definitions

- [x] If the same fully qualified class name appears in two modules, or if a class defines two methods with identical signatures, there was no error code. *(fixed 0.8.24: compiler.md § Duplicate definitions — E041 and E042 added; see also II-17)*

### IV-8. Import name conflict — no error code or resolution rule

- [x] When an import would create a duplicate unqualified name (e.g. `use system.Out;` in a file defining class `Out`, or in a namespace that already has `Out`), the spec said "aliases must be unique" but did not define the compiler behavior or error code. *(fixed 0.8.32: specs.md § Import rules — explicit "no duplicate unqualified names" rule; compiler.md § Import name resolution — E043 added)*

---

## V. Standard library issues (8 resolved)

### V-1. `system.Bool` — no `parse` method

- [x] `system.Int` has `parse`, `system.Float` has `parse`, but `system.Bool` had only `toString`. There was no `parse(string s)` for converting `"true"` / `"false"` to `bool`. This was an API asymmetry. *(fixed 0.8.28: stdlib.md § system.Bool — added parse; 0.8.29: renamed parseInt/parseFloat/parseBool to parse across Int, Float, Bool)*

### V-2. `system.String` — `trim` and `split` were static but peers are instance methods

- [x] String instance methods (`length`, `charAt`, `substring`, `indexOf`, `contains`, `replace`, `toUpperCase`, `toLowerCase`, `startsWith`, `endsWith`) are called on string values: `text.length()`. But `trim` and `split` were static methods: `system.String.trim(text)` and `system.String.split(s, delim)`. This asymmetry forced users to remember which methods are where. *(fixed 0.8.30: stdlib.md § system.String — trim() and split(delimiter) are now instance methods; static forms trim(s) and split(s, delim) kept for flexibility)*

### V-3. `system.List<T>` — no positional `remove` or `contains`

- [x] `system.List<T>` provides `pushBack`, `pushFront`, `popBack`, `popFront`, `get`, `set`, `add`, `size`, and for-each support. But there was no `remove(int index)` for arbitrary-position removal and no `contains(T value)` for searching. *(fixed 0.8.31: stdlib.md § system.List — added remove(int index) and contains(T value))*

### V-4. `system.thread.Thread` — no `isAlive()` method

- [x] There is no way to check if a thread is still running without blocking on `join()`. An `isAlive()` method (returning `bool`) is standard in most threading APIs. *(fixed 0.8.33: stdlib.md § system.thread.Thread — added isAlive())*

### V-5. Thread safety of collections not documented

- [x] `system.List<T>` and `system.Map<K, V>` are heap objects shared across threads (vm.md § Threading model). The spec does not state whether they are thread-safe or require external synchronization. This must be documented (most likely: *not* thread-safe — caller must use a `Mutex`). *(fixed 0.8.34: stdlib.md § system.List, § system.Map — explicit "not thread-safe" note; caller must use Mutex for concurrent access)*

### V-6. `system.io.File.open` — no file mode parameter

- [x] `File.open(string path)` opens a file for "reading and/or writing." There is no parameter to control the open mode (read-only, write-only, append, truncate, create). This makes the API unsuitable for many use cases. *(fixed 0.8.35: stdlib.md § system.io.FileMode, § system.io.File — added FileMode enum and open(path, mode) overload)*

### V-7. `Stringable`, `Cloneable`, `ValueEquatable` not listed in stdlib

- [x] These three core interfaces are defined in **specs.md** (§ Extends/Implements, § Cloneable interface, § ValueEquatable interface) but never mentioned in **stdlib.md**. They are part of the language/runtime contract and should appear in stdlib.md (or at least be cross-referenced) since implementors need to know they are built-in. *(fixed 0.8.3: added § Core interfaces (built-in) in stdlib.md with cross-references)*

### V-8. No `StackOverflowException` in exception hierarchy

- [x] Infinite recursion will exhaust the call stack. There was no `StackOverflowException` in the exception hierarchy. *(fixed 0.8.24: added `StackOverflowException extends RuntimeException` in specs.md § Exception class hierarchy, stdlib.md exceptions table, and vm.md § Call frame)*

---

## VI. Under-specified semantics (1 resolved)

### VI-9. `system.ps.Process.exit` — control flow impact

- [x] `Process.exit(int code)` is documented as *"Does not return."* But the compiler didn't know the code after it is unreachable. *(fixed 0.8.24: compiler.md § Terminal statements — `return`, `throw`, `Process.exit()` defined as terminal; stdlib.md updated)*

---

## VII. Documentation and editorial errors (6 resolved)

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

## VIII. Security-related specification gaps (1 resolved)

### VIII-8. Read/write after close behavior unspecified

- [x] **stdlib.md § system.io.FileHandle, system.net.TcpStream, system.net.UdpSocket** — `close()` is idempotent, but the behavior of `read()`, `write()`, `readLine()`, `flush()` after `close()` is not specified. Should throw `IOException`. *[SEC-11]* *(fixed 0.8.23)*
