# NL Specification — Security Audit

Audit performed on **2026-03-04**, against spec version **0.8.21**.

This report identifies potential security vulnerabilities, weaknesses, and missing hardening measures in the NL language specification. Each finding is mapped to a [CWE](https://cwe.mitre.org/) identifier and references known CVE patterns where applicable. Findings are classified by severity (Critical, High, Medium, Low, Informational).

---

## Summary

| Severity | Count | Description |
|----------|-------|-------------|
| [Critical](#critical) | 3 | Exploitable vulnerabilities that can lead to arbitrary code execution, complete system compromise |
| [High](#high) | 7 | Significant vulnerabilities: data corruption, denial of service, information disclosure |
| [Medium](#medium) | 8 | Design weaknesses that require specific conditions to exploit |
| [Low](#low) | 5 | Minor issues, defense-in-depth gaps |
| [Informational](#informational) | 3 | Best-practice recommendations |

---

## Critical

### SEC-01. Command Injection via `system.ps.Process.run(string)`

**CWE:** [CWE-78 — OS Command Injection](https://cwe.mitre.org/data/definitions/78.html)
**CVE pattern:** CVE-2021-44228 (Log4Shell class — unsanitized user input reaches execution surface), CVE-2014-6271 (Shellshock — shell interpretation of unsanitized input)

**Location:** stdlib.md § system.ps.Process

**Description:** `Process.run(string command)` runs a command *"via the platform shell"*. The spec provides no guidance on input sanitization, shell escaping, or the risks of passing user-controlled strings. The `string[]` overload (`run(string[] args)`) is safer (avoids shell interpretation), but the single-string overload is a direct command injection vector.

```nl
string userInput = system.In.readLine() ?? "";
auto result = system.ps.Process.run("grep " + userInput + " /etc/passwd");
// userInput = "; rm -rf /" → shell injection
```

**Recommendation:**
1. Add a **security warning** in stdlib.md documenting that `run(string command)` passes input to the platform shell and that user-controlled values must never be interpolated without escaping.
2. Recommend the `run(string[] args)` overload as the safe alternative (bypasses shell interpretation).
3. Consider deprecating or removing the single-string overload entirely — most modern language specs (Go's `os/exec`, Rust's `std::process::Command`) avoid it.

---

### SEC-02. Path Traversal — File System APIs lack sanitization

**CWE:** [CWE-22 — Path Traversal](https://cwe.mitre.org/data/definitions/22.html)
**CVE pattern:** CVE-2021-21972, CVE-2019-11510 (arbitrary file read via path traversal), CVE-2023-34362 (MOVEit)

**Location:** stdlib.md § system.io.File, system.io.Directory, system.io.Path

**Description:** None of the file system APIs (`File.open`, `File.readAllText`, `File.writeAllText`, `File.glob`, `Directory.create`, `Directory.remove`, `Process.setCwd`) perform path sanitization. There is no restriction on absolute paths, `..` traversal, or symlink following. `Path.normalize` resolves `.` and `..` but it is a utility — the spec does not require its use before file operations.

```nl
string filename = system.In.readLine() ?? "";
string content = system.io.File.readAllText(filename);
// filename = "../../../../etc/shadow" → reads sensitive system file
```

**Recommendation:**
1. Document that file system APIs operate with the privileges of the NL process and that path validation is the caller's responsibility.
2. Add a security section in stdlib.md warning about path traversal risks.
3. Consider specifying an optional sandboxing mechanism (a "base directory" that restricts all I/O to a subtree) for future milestones.

---

### SEC-03. No Module Integrity Verification — Unsigned Bytecode

**CWE:** [CWE-494 — Download of Code Without Integrity Check](https://cwe.mitre.org/data/definitions/494.html)
**CVE pattern:** CVE-2020-1472 (Zerologon-class: trust without verification), supply-chain attacks (SolarWinds/CVE-2020-10148)

**Location:** vm.md § Module format

**Description:** The `.nlm` module format has only a 4-byte magic number (`0x4E4C4D00`) and a version field. There is no:
- Cryptographic hash or checksum for integrity verification.
- Code signing mechanism (digital signature).
- Origin verification for loaded modules.

A malicious actor who can place a modified `.nlm` file in the module path can execute arbitrary bytecode. The `--module-path` VM option compounds the risk by allowing additional module search directories.

**Recommendation:**
1. Add an optional `integrity` section to the module format (e.g., SHA-256 hash of the code sections).
2. Define a code signing specification for production deployments.
3. Document the trust model: who is trusted to produce `.nlm` files, and what verification (if any) the VM performs at load time.

---

## High

### SEC-04. Denial of Service — No Resource Limits

**CWE:** [CWE-400 — Uncontrolled Resource Consumption](https://cwe.mitre.org/data/definitions/400.html), [CWE-770 — Allocation of Resources Without Limits](https://cwe.mitre.org/data/definitions/770.html)
**CVE pattern:** CVE-2018-1000861 (Jenkins — unbounded resource allocation), CVE-2022-42889 (Text4Shell)

**Location:** vm.md (entire execution model), stdlib.md (I/O, networking, threading)

**Description:** The specification defines no limits on:

| Resource | Missing limit |
|----------|---------------|
| Call stack depth | No maximum depth → infinite recursion consumes all memory (no `StackOverflowException`; see coherence V-8) |
| Heap size | No maximum allocation → `new T[Integer.MAX_VALUE]` or unbounded list growth |
| Thread count | No cap on `system.thread.Thread` creation |
| Open file handles | No per-process FD limit |
| Socket connections | No connection pool limit or timeout |
| String length | No maximum → `string` concatenation in a loop |
| Read buffer size | `File.readAllText` on a multi-GB file |
| Network timeouts | `TcpListener.accept()`, `TcpStream.read()`, `Http.get()` can block indefinitely |
| Regex execution time | No protection against catastrophic backtracking (ReDoS) |

**Recommendation:**
1. Add a StackOverflowException (or equivalent) to the exception hierarchy (reinforces coherence V-8).
2. Specify that implementations **should** provide configurable resource limits (stack depth, heap size, thread count).
3. Add timeout parameters or overloads for blocking I/O and network operations.
4. Document ReDoS risks in `system.text.Regex` and recommend implementation-level safeguards (regex compilation limits, execution timeouts).

---

### SEC-05. Parsing RuntimeExceptions Bypass Checked Exception Handling

**CWE:** [CWE-755 — Improper Handling of Exceptional Conditions](https://cwe.mitre.org/data/definitions/755.html)

**Location:** stdlib.md § system.Int, system.Float; compiler.md § Checked exception propagation

**Description:** `parseInt` and `parseFloat` throw `NumberFormatException`, a `RuntimeException`. Since runtime exceptions do not require `throws` declarations or `try/catch`, programs that parse untrusted input have no compile-time safety net:

```nl
string input = args[1];
int n = system.Int.parseInt(input);  // no try/catch required
// if input = "abc" → unhandled NumberFormatException → process crash
```

This is a denial-of-service vector: any NL program that parses external input (CLI args, files, network data) without explicit `try/catch` will crash on malformed input. The spec's own example (specs.md § Entry point, line 2405) parses `args[2]` and `args[3]` without error handling.

**Recommendation:**
1. Add `tryParseInt(string s) : int|null` and `tryParseFloat(string s) : float|null` methods that return `null` on invalid input (safe-by-default API).
2. Document that `parseInt`/`parseFloat` will crash on invalid input if not wrapped in `try/catch`.
3. Fix the entry point example in specs.md to demonstrate proper error handling.

---

### SEC-06. Race Conditions — Thread Safety Unspecified

**CWE:** [CWE-362 — Race Condition](https://cwe.mitre.org/data/definitions/362.html), [CWE-567 — Unsynchronized Access to Shared Data](https://cwe.mitre.org/data/definitions/567.html)
**CVE pattern:** CVE-2019-1040 (NTLM relay — race in authentication), CVE-2016-5195 (Dirty COW)

**Location:** vm.md § Threading model; stdlib.md § system.List, system.Map, system.Env

**Description:** The spec states that heap objects (including static fields) are shared across threads, but:
- `system.List<T>` and `system.Map<K, V>` thread safety is not documented (coherence V-5).
- `system.Env.set()` / `system.Env.remove()` can be called from any thread — concurrent modification of the process environment is undefined behavior on most platforms.
- No volatile/atomic semantics exist. Unsynchronized access to shared fields "may produce stale or inconsistent values."
- The Mutex API has no RAII-style guard (no `try-with-resources`), making it easy to forget `unlock()` after exceptions.

**Recommendation:**
1. State explicitly that `system.List`, `system.Map`, and `system.Env` are **not** thread-safe (concurrent access requires `Mutex`).
2. Document that `Env.set()`/`Env.remove()` from multiple threads is undefined behavior.
3. Consider adding a `Mutex.withLock(() => void)` convenience method that guarantees unlock.
4. Plan for volatile/atomic field semantics in a future spec version.

---

### SEC-07. Information Disclosure via Process and Environment APIs

**CWE:** [CWE-200 — Exposure of Sensitive Information](https://cwe.mitre.org/data/definitions/200.html), [CWE-214 — Invocation of Process Using Visible Sensitive Information](https://cwe.mitre.org/data/definitions/214.html)

**Location:** stdlib.md § system.ps.Process, system.Env

**Description:**
- `Process.list()` reveals all processes visible to the current process (PIDs, commands, arguments, user names). On many platforms, command-line arguments may contain passwords, tokens, or connection strings.
- `Env.list()` returns all environment variable names; `Env.get()` retrieves values. Environment variables frequently contain API keys, database credentials, and secrets (`AWS_SECRET_ACCESS_KEY`, `DATABASE_URL`, etc.).
- Stack traces in exceptions include file paths and line numbers, which reveal internal project structure.

**Recommendation:**
1. Document that `Process.list()` and `Env.list()`/`Env.get()` expose sensitive information and should not be used to display data to untrusted users.
2. Consider a filtering API for `Process.list()` (e.g., filter to own process tree).
3. Recommend that stack traces not be displayed to end users in production.

---

### SEC-08. No TLS Configuration for Network APIs

**CWE:** [CWE-319 — Cleartext Transmission of Sensitive Information](https://cwe.mitre.org/data/definitions/319.html), [CWE-295 — Improper Certificate Validation](https://cwe.mitre.org/data/definitions/295.html)
**CVE pattern:** CVE-2014-0160 (Heartbleed), CVE-2015-0204 (FREAK)

**Location:** stdlib.md § system.net.Http, system.net.TcpStream

**Description:**
- `Http.get(string url)` accepts `https://` URLs but no TLS configuration is specified (certificate validation, TLS version, cipher suites).
- `TcpStream` provides raw TCP only — no TLS/SSL wrapper.
- No mechanism for certificate pinning or custom trust stores.
- Whether the runtime validates server certificates by default is unspecified. An implementation that skips validation would be vulnerable to man-in-the-middle attacks.

**Recommendation:**
1. Specify that `Http.get`/`Http.post` on `https://` URLs **must** validate server certificates by default.
2. Add TLS-related options (at minimum: verify/skip verify flag, with verify as default).
3. Consider a `system.net.TlsStream` wrapper for `TcpStream` in a future milestone.

---

### SEC-09. Weak / Non-Cryptographic Randomness

**CWE:** [CWE-330 — Use of Insufficiently Random Values](https://cwe.mitre.org/data/definitions/330.html), [CWE-338 — Use of Cryptographically Weak PRNG](https://cwe.mitre.org/data/definitions/338.html)
**CVE pattern:** CVE-2008-0166 (Debian OpenSSL — weak key generation)

**Location:** stdlib.md § system.Random, system.Uuid

**Description:**
- `system.Random` is described as "pseudo-random" with an "implementation-defined" seed. It is suitable for games and simulations but **not** for security-sensitive operations (session tokens, nonces, keys).
- `system.Uuid.random()` does not specify the UUID version. A V1 UUID reveals the MAC address and timestamp; only V4 (random) is safe for unpredictable identifiers. The randomness source is unspecified.
- There is no cryptographically secure random number generator (CSPRNG) in the standard library.

**Recommendation:**
1. Add a `system.SecureRandom` class backed by the OS CSPRNG (e.g., `/dev/urandom`, `BCryptGenRandom`).
2. Specify that `system.Uuid.random()` generates UUID v4 using the CSPRNG.
3. Document that `system.Random` is **not** suitable for security purposes.

---

### SEC-10. Integer Overflow Wraps Silently

**CWE:** [CWE-190 — Integer Overflow](https://cwe.mitre.org/data/definitions/190.html), [CWE-191 — Integer Underflow](https://cwe.mitre.org/data/definitions/191.html)
**CVE pattern:** CVE-2021-22555 (Linux Netfilter integer overflow), CVE-2014-1266 (Apple goto fail — related to arithmetic errors)

**Location:** vm.md § Integer arithmetic

**Description:** Integer arithmetic (IADD, ISUB, IMUL) wraps on overflow (two's complement). This is standard behavior but a well-known source of security bugs:
- Array size calculations: `int size = width * height; new byte[size];` — if `width * height` overflows, the allocated buffer is smaller than expected.
- The spec's own example `numbers.sort((int a, int b) => a - b)` (specs.md line 304) will produce incorrect ordering when `a` and `b` have opposite signs and large magnitude (e.g., `a = INT_MAX`, `b = -1` → `a - b` overflows).
- Financial calculations, bounds checking, and any arithmetic on untrusted input are vulnerable.

`specs.md` does not mention integer overflow behavior at all — a developer might not know about wrapping semantics.

**Recommendation:**
1. Document integer overflow behavior in specs.md § Native types (wrapping two's complement for `int`; implementation-defined for `byte`).
2. Consider adding checked arithmetic methods: `system.Int.addExact(int, int) throws ArithmeticException` (as in Java 8+ `Math.addExact`).
3. Fix the comparator example to use `a < b ? -1 : (a > b ? 1 : 0)` instead of `a - b`.

---

## Medium

### SEC-11. Read/Write After Close on I/O Handles

**CWE:** [CWE-416 — Use After Free](https://cwe.mitre.org/data/definitions/416.html) (analogous pattern)

**Location:** stdlib.md § system.io.FileHandle, system.net.TcpStream, system.net.UdpSocket

**Description:** `close()` is documented as idempotent, but the behavior of `read()`, `write()`, `readLine()`, `flush()` after `close()` is unspecified. Depending on the implementation, this could produce corrupted data, silently succeed (writing to a reused file descriptor), or crash.

**Recommendation:** Specify that calling I/O methods on a closed handle throws `IOException` (or a dedicated `ClosedResourceException`). *(Specified in stdlib.md 0.8.23: FileHandle, TcpStream, UdpSocket — read/write/flush/send/receive on closed handle throw IOException.)*

---

### SEC-12. No RAII / Deterministic Cleanup for Resources

**CWE:** [CWE-404 — Improper Resource Shutdown or Release](https://cwe.mitre.org/data/definitions/404.html), [CWE-772 — Missing Release of Resource after Effective Lifetime](https://cwe.mitre.org/data/definitions/772.html)

**Location:** specs.md § Planned; vm.md § Garbage collection contract

**Description:** The spec explicitly notes that try-with-resources / RAII is **not** yet specified (specs.md § Planned). File handles, sockets, and mutexes must be closed manually in `finally` blocks. The GC may call destructors "promptly" but is not required to. This leads to:
- File descriptor leaks (can exhaust OS limits → DoS).
- Mutex leaks (deadlocks if `unlock()` is skipped after an exception).
- Socket leaks (port exhaustion → network DoS).

**Recommendation:** Prioritize a `try-with-resources` or `using` construct in a future spec version. Until then, add strong warnings in stdlib.md about always closing resources in `finally` blocks.

---

### SEC-13. SSRF via Network APIs

**CWE:** [CWE-918 — Server-Side Request Forgery](https://cwe.mitre.org/data/definitions/918.html)
**CVE pattern:** CVE-2021-26855 (ProxyLogon/Exchange SSRF)

**Location:** stdlib.md § system.net.Http, system.net.TcpStream

**Description:** `Http.get(string url)` and `TcpStream.connect(string host, int port)` accept arbitrary URLs and host/port combinations with no restrictions. If an NL application takes a URL from user input and fetches it, an attacker can:
- Probe internal services (`http://169.254.169.254/latest/meta-data/` on cloud instances).
- Port-scan internal networks.
- Access localhost-only services.

**Recommendation:** Document SSRF risks. Recommend that applications validate and whitelist URLs/hosts before making network requests. Consider an allow-list mechanism for outbound connections.

---

### SEC-14. Sensitive Data in Immutable Strings

**CWE:** [CWE-316 — Cleartext Storage of Sensitive Information in Memory](https://cwe.mitre.org/data/definitions/316.html)

**Location:** vm.md § String representation; optimizations.md § String interning

**Description:** Strings are immutable heap objects. There is no mechanism to:
- Zero out a string's contents after use (for passwords, tokens, keys).
- Prevent string interning of sensitive values (interned strings persist for the process lifetime).
- Mark a string as "sensitive" to avoid it appearing in core dumps, logs, or stack traces.

Passwords stored as `string` remain in memory until the GC reclaims the object (timing undefined). String interning may keep them alive even longer.

**Recommendation:**
1. Consider a `system.SecureString` or `byte[]`-based alternative for sensitive data that can be explicitly zeroed.
2. Document that `string` should not be used for long-lived secrets.

---

### SEC-15. Float-to-Int Conversion — Undefined vs Clamped Behavior

**CWE:** [CWE-681 — Incorrect Conversion between Numeric Types](https://cwe.mitre.org/data/definitions/681.html)

**Location:** specs.md § Type conversions; vm.md § F2I (also coherence IV-5)

**Description:** specs.md says values outside `int` range have *"undefined behavior (or implementation-defined)"*, while vm.md specifies *"clamped to INT_MIN/INT_MAX"*. Undefined behavior in a type conversion is a classic security risk — it means an attacker-controlled float could produce arbitrary integer values in some implementations, breaking bounds checks or other arithmetic guards.

**Recommendation:** Resolve in favor of defined behavior (clamping). Align specs.md with vm.md. Undefined behavior in numeric conversions should be eliminated from the spec.

---

### SEC-16. Buffer Over-Read in Byte Array I/O

**CWE:** [CWE-125 — Out-of-bounds Read](https://cwe.mitre.org/data/definitions/125.html), [CWE-787 — Out-of-bounds Write](https://cwe.mitre.org/data/definitions/787.html)

**Location:** stdlib.md § system.io.FileHandle, system.net.TcpStream, system.net.UdpSocket

**Description:** The `read(byte[] buffer, int offset, int length)` and `write(byte[] data, int offset, int length)` methods accept arbitrary `offset` and `length` values. The spec does not specify what happens when:
- `offset < 0` or `length < 0`
- `offset + length > buffer.length()`
- `offset` or `length` are `INT_MAX` (causing `offset + length` to overflow)

In a native implementation, incorrect bounds could cause buffer over-reads (information disclosure) or over-writes (memory corruption).

**Recommendation:** Specify that `read`/`write` **must** throw `IndexOutOfBoundsException` if `offset < 0 || length < 0 || offset + length > buffer.length()`. The bounds check must be performed before any native I/O operation.

---

### SEC-17. Readonly Bypass via VM Stack Trace Population

**CWE:** [CWE-471 — Modification of Assumed-Immutable Data](https://cwe.mitre.org/data/definitions/471.html)

**Location:** vm.md § Stack trace construction; specs.md § Exception class hierarchy (also coherence IV-7)

**Description:** `Exception` is declared `class readonly`, meaning all fields are immutable after construction. However, the `stackTrace` field is populated by the VM **after** the constructor runs. This requires the VM to bypass readonly enforcement — a violation of the language's own immutability guarantees. If the VM has a mechanism to bypass readonly, that mechanism could be exploited (or trigger implementation bugs) in other contexts.

**Recommendation:** Either (a) populate `stackTrace` inside the constructor via a native call (the constructor is the last place where readonly fields can be assigned), or (b) make `stackTrace` a VM-internal data structure accessed via a native method (`getStackTrace()`) rather than a public field.

---

### SEC-18. Environment Variable Injection

**CWE:** [CWE-426 — Untrusted Search Path](https://cwe.mitre.org/data/definitions/426.html)

**Location:** stdlib.md § system.Env

**Description:** `system.Env.set(string name, string value)` can set arbitrary environment variables, including security-sensitive ones like `PATH`, `LD_PRELOAD`, `LD_LIBRARY_PATH`, `HOME`, or `http_proxy`. When combined with `Process.run()`, this allows:
- DLL/shared library injection (modifying `PATH` or `LD_PRELOAD` before launching a subprocess).
- Proxy hijacking (setting `http_proxy` to redirect network traffic).

**Recommendation:** Document the risks. In security-sensitive contexts, implementations **should** consider restricting which environment variables can be modified (or at least warn about `LD_PRELOAD`, `PATH`, etc.).

---

## Low

### SEC-19. No Duplicate Method/Class Detection at VM Level

**CWE:** [CWE-694 — Use of Multiple Resources with Duplicate Identifier](https://cwe.mitre.org/data/definitions/694.html)

**Location:** compiler.md (also coherence IV-6)

**Description:** If two modules define the same fully qualified class name, or a class defines two methods with identical signatures, there is no specified behavior. A malicious module could shadow a legitimate class. This is noted in coherence IV-6 but has security implications: module shadowing can redirect method calls to attacker-controlled code.

**Recommendation:** Add error codes for duplicate class and duplicate method detection, both at compile time and at VM link time.

---

### SEC-20. `Process.exit()` as Terminal Statement Not Specified

**CWE:** [CWE-705 — Incorrect Control Flow Scoping](https://cwe.mitre.org/data/definitions/705.html)

**Location:** coherence VI-9

**Description:** `Process.exit(int code)` is documented as "does not return" but is not classified as a terminal statement (like `throw`). This means code after `exit()` may be reached in the compiler's analysis, potentially leading to uninitialized variable use or missing return value errors being incorrectly reported — or conversely, real issues being masked.

**Recommendation:** Classify `Process.exit()` as a terminal statement (equivalent to `throw`) for control-flow analysis purposes.

---

### SEC-21. Destructor Exception Swallowing

**CWE:** [CWE-390 — Detection of Error Condition Without Action](https://cwe.mitre.org/data/definitions/390.html)

**Location:** vm.md § Garbage collection contract

**Description:** Destructors that throw have their exceptions *"silently discarded."* If a destructor performs security-critical cleanup (wiping memory, revoking tokens, releasing locks), a thrown exception means the cleanup failed — and nobody knows.

**Recommendation:** Specify that destructor exceptions are logged to stderr (or an implementation-defined error log) before being discarded, so security-relevant failures are at least observable.

---

### SEC-22. No `instanceof` in Language Spec

**CWE:** [CWE-843 — Access of Resource Using Incompatible Type](https://cwe.mitre.org/data/definitions/843.html)

**Location:** coherence II-1

**Description:** The VM defines an `INSTANCEOF` opcode, but specs.md does not define the `instanceof` keyword or expression. Without a user-facing type test, developers must use unsafe downcasts (which throw `InvalidCastException`, a `RuntimeException` that doesn't require handling). Safe type narrowing requires `instanceof`.

**Recommendation:** Define `instanceof` as a binary expression (`expr instanceof ClassName → bool`) in specs.md.

---

### SEC-23. Regex Without Anchoring or Escaping Utilities

**CWE:** [CWE-185 — Incorrect Regular Expression](https://cwe.mitre.org/data/definitions/185.html)

**Location:** stdlib.md § system.text.Regex, system.io.Grep

**Description:** The spec provides `Regex.match`, `Regex.replace`, and `Grep.search` but no utility to:
- Escape a string for use as a literal pattern (`Regex.escape(string) : string`).
- Anchor a match (is `match` full-match or partial?).

If `match("user_input", data)` is partial-match, user-controlled patterns may match unintended content. If `user_input` contains regex metacharacters, the behavior is unpredictable.

**Recommendation:**
1. Add a `Regex.escape(string) : string` method to safely use user input in patterns.
2. Clarify whether `Regex.match()` is a full-match or partial-match operation.

---

## Informational

### SEC-24. No Security Namespace or Security Guidelines

**Description:** The standard library covers I/O, networking, threads, and text processing but provides no security-oriented APIs:
- No hashing (SHA-256, HMAC).
- No encryption/decryption (AES, RSA).
- No CSPRNG (see SEC-09).
- No secure comparison (`constantTimeEquals` to prevent timing attacks).

**Recommendation:** Plan a `system.security` or `system.crypto` namespace for future milestones. At minimum: CSPRNG, cryptographic hashing, and constant-time comparison.

---

### SEC-25. No Sandboxing or Capability Model

**Description:** Any NL program has full access to the file system, network, process table, and environment variables. There is no mechanism to restrict capabilities (e.g., "this module can only read files in `/data/`"). Modern runtimes (Deno, Java SecurityManager [deprecated but instructive], WebAssembly) provide capability-based security.

**Recommendation:** Consider a permission model for future milestones — at minimum, a way to disable dangerous APIs (process execution, environment modification, network access) for untrusted code.

---

### SEC-26. Spec Examples Demonstrate Insecure Patterns

**Description:** Several spec examples demonstrate patterns that would be insecure in production:
- specs.md § Entry point (line 2405): parses `args[2]`, `args[3]` with `parseInt` — no error handling.
- stdlib.md § TcpListener example: accepts connections in a tight loop with no rate limiting, no client authentication.
- stdlib.md § File.open example: opens a file without checking permissions or validating the path.

**Recommendation:** Add security annotations or comments to examples that handle external input, or provide both "simple" and "production" versions of examples.

---

## Cross-Reference with Existing Coherence Items

Several coherence.md items have direct security implications:

| Coherence Item | Security Finding |
|----------------|-----------------|
| I-5 (`throws` on RuntimeException) | Relates to SEC-05 (parsing exceptions bypass checked handling) |
| II-1 (no `instanceof`) | SEC-22 (forces unsafe downcasts) |
| IV-5 (F2I undefined vs clamped) | SEC-15 (undefined behavior in type conversion) |
| IV-6 (no duplicate method/class error) | SEC-19 (module shadowing) |
| IV-7 (readonly bypass for stackTrace) | SEC-17 (immutability violation) |
| V-5 (thread safety undocumented) | SEC-06 (race conditions) |
| V-8 (no StackOverflowException) | SEC-04 (denial of service) |
| VI-9 (Process.exit control flow) | SEC-20 (incorrect control-flow analysis) |

---

## Prioritized Recommendations

### Immediate (before 1.0)

1. **SEC-01**: Add security warning for `Process.run(string)` — command injection.
2. **SEC-02**: Document path traversal risks in file system APIs.
3. **SEC-05**: Add `tryParseInt`/`tryParseFloat` safe parsing methods.
4. **SEC-06**: Declare thread safety posture of stdlib collections and Env.
5. **SEC-10**: Document integer overflow behavior in specs.md.
6. **SEC-15**: Resolve F2I undefined behavior (align specs.md with vm.md).
7. **SEC-16**: Specify bounds checking for byte array I/O methods.
8. **SEC-17**: Resolve readonly bypass for Exception.stackTrace.

### Short-term (1.x)

9. **SEC-03**: Module integrity (hash in module format).
10. **SEC-04**: StackOverflowException + configurable resource limits.
11. **SEC-08**: Specify TLS certificate validation requirements.
12. **SEC-09**: Add `system.SecureRandom` CSPRNG.
13. ~~**SEC-11**: Specify IOException on read/write after close.~~ *(fixed 0.8.23)*
14. **SEC-12**: Design try-with-resources or RAII mechanism.

### Long-term (2.x+)

15. **SEC-24**: `system.crypto` namespace.
16. **SEC-25**: Capability-based sandboxing model.
17. **SEC-14**: Secure string handling.
