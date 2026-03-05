# NL Specification — Coherence Tracker

This document lists all known inconsistencies, errors, omissions, and gaps across the NL specification
documents (`specs.md`, `stdlib.md`, `compiler.md`, `vm.md`, `optimizations.md`, `milestones.md`, `tests.md`, `README.md`).
Each item has a checkbox to track resolution.

Audit performed on **2026-03-03**, against spec version **0.8.1**.

**Resolved items archived:** [archives/coherence_closed_20260305.md](archives/coherence_closed_20260305.md) (49 items, 2026-03-05)

---

## Summary (open items)

| Category | Open | Description |
|----------|------|-------------|
| [II. Language specification omissions](#ii-language-specification-omissions) | 6 | Features referenced or implied but never formally defined in specs.md |
| [IV. VM and compiler specification gaps](#iv-vm-and-compiler-specification-gaps) | 5 | Missing pieces in vm.md or compiler.md |
| [VI. Under-specified semantics](#vi-under-specified-semantics) | 8 | Defined but incomplete — a compiler/VM implementor cannot proceed without guessing |
| [VIII. Security-related specification gaps](#viii-security-related-specification-gaps) | 9 | Missing security hardening, unsafe APIs, unspecified safety behavior — see [security-audit.md](security-audit.md) |

**Total open: 28**

---

## II. Language specification omissions

### II-1. No `instanceof` keyword or expression

- [ ] **vm.md** defines an `INSTANCEOF` opcode (line 472). **milestones.md** (line 38) lists `instanceof` as an expression. But **specs.md** does not define an `instanceof` keyword, does not list it in the keywords table, and does not describe any expression syntax for runtime type testing. A compiler implementor cannot generate `INSTANCEOF` without knowing the source-level syntax. Suggested: add `instanceof` as a binary expression `expr instanceof ClassName` → `bool`.

### II-2. No complete operator precedence table

- [ ] **specs.md § Operator precedence and usage** gives only partial relative precedence for `??`, `?:`, and `? :`. A full operator precedence table (from highest to lowest) is missing. Without it, expressions mixing multiple operators have ambiguous parsing.

### II-8. Interface extending interface

- [ ] Can interfaces extend other interfaces? E.g. `interface Closeable extends Disposable`. The spec shows classes extending classes and classes implementing interfaces, but never interface-to-interface inheritance.

### II-9. Constructor chaining (`this(…)`)

- [ ] In Java, a constructor can call another overloaded constructor with `this(args)`. The NL spec only defines `super(args)` for parent constructor calls. Constructor chaining within the same class is not mentioned.

### II-11. `match` expression — `default` clause mandatory?

- [ ] If no arm of a `match` expression matches and there is no `default`, what happens? Is `default` mandatory? Unreachable `default` for exhaustive matches? The spec is silent.

### II-16. `Self` in interface context

- [ ] `Cloneable` defines `public Self clone()`. In the interface, `Self` refers to "the implementing class" — a covariant return type. This semantic is used but never formally defined. What does `Self` resolve to inside an interface body? How does the compiler handle it?

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

### IV-7. Exception `readonly` vs VM `stackTrace` assignment

- [ ] The exception hierarchy declares `class readonly Exception` with field `public ExecutionPoint[] stackTrace`. In a readonly class, fields can only be assigned inside `construct`. But the stack trace is populated by the VM when the exception object is created — **after** the constructor runs. This requires the VM to bypass readonly enforcement for this specific field, which contradicts the spec. Either `stackTrace` should not be a regular field (use a native/internal mechanism), or the readonly rule needs an exception for VM-populated fields.

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

### VIII-9. `Regex.match` — full vs partial match unspecified

- [ ] **stdlib.md § system.text.Regex** — `Regex.match(string pattern, string input)` returns `bool`, but it is not specified whether this is a **full match** (the entire input must match the pattern) or a **partial match** (the pattern must be found somewhere in the input). No `Regex.escape()` utility exists for safely using user input in patterns. *[SEC-23]*

### VIII-10. Module format — no integrity verification

- [ ] **vm.md § Module format** — The `.nlm` format has a 4-byte magic number but no hash, checksum, or digital signature. A modified module can be loaded without detection. *[SEC-03]*

---

## Previous archives

- [archives/coherence_closed_20260303.md](archives/coherence_closed_20260303.md) — First coherence pass (20 items, all resolved in versions 0.3.1–0.8.1)
