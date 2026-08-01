# NL Standard Library (System API)

This document describes the virtual standard library provided by the NL runtime for system interaction: standard streams (out, err, in), parsing, file system access (including glob, directories, paths), network sockets (TCP, UDP) and HTTP, threads and synchronization (Mutex, Semaphore), date/time with timezone, grep-style text search, environment variables, process listing and subprocess execution, string and regex utilities, random and UUID, encoding and base64, JSON parsing and serialization, SQL database connectivity (SQLite, MySQL). All types live in the **`system`**, **`system.io`**, **`system.net`**, **`system.thread`**, **`system.time`**, **`system.ps`**, **`system.text`**, **`system.text.json`**, **`system.db`**, **`system.db.sqlite`**, and **`system.db.mysql`** namespaces and are available without explicit import in user code (built-in bindings). Return types written as **`T|null`** (or other union types like **`Type1|Type2|null`**) denote nullable or union types as defined in the language specification (see [specs.md](specs.md)#union-types-and-explicit-nullable).

---

## Summary

* [Namespaces](#namespaces)
* [Result types](#result-types)
* [Arrays (built-in)](#arrays-built-in)
* [Core interfaces (built-in)](#core-interfaces-built-in)
* [system.Out (stdout)](#systemout-stdout)
* [system.Err (stderr)](#systemerr-stderr)
* [system.In (stdin)](#systemin-stdin)
* [system.Int](#systemint)
* [system.Float](#systemfloat)
* [system.Bool](#systembool)
* [system.String](#systemstring)
* [system.Random](#systemrandom)
* [system.SecureRandom](#systemsecurerandom)
* [system.Uuid](#systemuuid)
* [system.List](#systemlist)
* [system.Map](#systemmap)
* [system.Env](#systemenv)
* [system.io.FileMode](#systemiofilemode)
* [system.io.File](#systemiofile)
* [system.io.File (glob)](#systemiofile-glob)
* [system.io.FileHandle](#systemiofilehandle)
* [system.io.Directory](#systemiodirectory)
* [system.io.Path](#systemiopath)
* [system.io.Grep](#systemiogrep)
* [system.net.TcpListener](#systemnettcplistener)
* [system.net.TcpStream](#systemnettcpstream)
* [system.net.UdpSocket](#systemnetudpsocket)
* [system.net.Http](#systemnethttp)
* [system.thread.Thread](#systemthreadthread)
* [system.thread.Mutex](#systemthreadmutex)
* [system.thread.Semaphore](#systemthreadsemaphore)
* [system.time.DateTime](#systemtimedatetime)
* [system.time.TimeZone](#systemtimetimezone)
* [system.ps](#systemps)
* [system.text.Regex](#systemtextregex)
* [system.text.Encoding](#systemtextencoding)
* [system.text.json](#systemtextjson)
* [system.db](#systemdb)
* [system.db.sqlite](#systemdbsqlite)
* [system.db.mysql](#systemdbmysql)
* [Exceptions](#exceptions)

---

## Namespaces

| Namespace   | Purpose                          |
|------------|-----------------------------------|
| `system`   | Standard streams (Out, Err, In), parsing and conversion (Int, Float, Bool), String, Random, SecureRandom, Uuid, Env, **List&lt;T&gt;**, **Map&lt;K,V&gt;**, **MapEntry&lt;K,V&gt;**; core interfaces (Stringable, Cloneable, ValueEquatable) |
| `system.io`| File system (File, FileHandle, FileMode, Directory, Path), glob, Grep |
| `system.net`| Network (TcpListener, TcpStream, UdpSocket, Http) |
| `system.thread`| Threads (Thread), synchronization (Mutex, Semaphore) |
| `system.time`| Date, time, timezone (DateTime, TimeZone) |
| `system.ps`| Process listing (Process.list, ProcessInfo), subprocess execution and current process (Process.run, Process.pid, Process.getCwd, Process.setCwd, Process.exit) |
| `system.text`| Regex, Encoding (UTF-8, base64) |
| `system.text.json`| JSON parsing and serialization (JsonValue and subclasses, Json) |
| `system.db`| SQL database connectivity — driver-agnostic types (Connection, PreparedStatement, ResultSet, Row, ColumnType) |
| `system.db.sqlite`| SQLite driver (Sqlite factory) |
| `system.db.mysql`| MySQL driver (Mysql factory) |

Classes in these namespaces are part of the language/runtime contract. User code may reference them by fully qualified name (e.g. `system.Out.print`) or after importing with `use system.Out;` (then `Out.print`).

---

## Result types

Several stdlib APIs return **result types**: runtime classes that carry structured data (public fields only). They are not meant to be constructed by user code; they are created and returned by the runtime when calling the corresponding methods. The following types are part of the language/runtime contract. An implementation must provide these types with at least the listed public fields.

### system.io.GrepMatch

Returned by **`system.io.Grep.search()`**. Represents one line matching a grep pattern.

| Field         | Type     | Description |
|---------------|----------|-------------|
| `path`        | `string` | Path of the file containing the match. |
| `lineNumber`  | `int`    | 1-based line number. |
| `line`        | `string` | Full text of the matching line. |

---

### system.ps.ProcessInfo

Returned by **`system.ps.Process.list()`**. Represents information about a process.

| Field     | Type        | Description |
|-----------|-------------|-------------|
| `pid`     | `int`       | Process ID. |
| `command` | `string`    | Command name or executable path. |
| `args`    | `string[]`  | Command-line arguments. |
| `user`    | `string\|null` | Owner or user name; `null` if not available or platform-specific. |

Additional platform-specific fields may be provided by the implementation.

---

### system.net.HttpResponse

Returned by **`system.net.Http.get()`** and **`system.net.Http.post()`**. Represents the HTTP response.

| Field        | Type        | Description |
|--------------|-------------|-------------|
| `statusCode` | `int`       | HTTP status code (e.g. 200, 404). |
| `body`       | `string`    | Response body (encoding is implementation-defined, e.g. UTF-8). |
| `headers`    | `string[]|null` | Optional: response headers (format implementation-defined). |

---

### system.text.RegexMatch

Returned by **`system.text.Regex.matchFirst()`**. Represents a single regex match with capture groups.

| Field       | Type       | Description |
|-------------|------------|-------------|
| `fullMatch` | `string`   | The full substring that matched the pattern. |
| `groups`    | `string[]` | Capture groups (index 0 may duplicate `fullMatch` or be the first group; exact layout is implementation-defined). |

---

### system.ps.ProcessResult

Returned by **`system.ps.Process.run()`**. Represents the outcome of a completed subprocess.

| Field      | Type     | Description |
|------------|----------|-------------|
| `exitCode` | `int`    | Process exit code. |
| `stdout`   | `string` | Standard output captured from the process. |
| `stderr`   | `string` | Standard error captured from the process. |

---

### system.MapEntry&lt;K, V&gt;

Returned by **`system.Map.entries()`** and used during **for-each iteration** over a map. Represents a single key-value pair. This is a native template result type — the runtime provides monomorphized instances (e.g. `MapEntry<string, int>`) as needed.

| Field   | Type | Description |
|---------|------|-------------|
| `key`   | `K`  | The key of the entry. |
| `value` | `V`  | The value associated with the key. |

---

## Arrays (built-in)

Array types `T[]` are part of the language (see [specs.md](specs.md)#arrays). Creation: `new T[]{ ... }` (initializer list) or `new T[n]` (fixed size). Access by index: `arr[i]` (get/set); out-of-range access throws **IndexOutOfBoundsException**. Built-in methods: **`length()`**, **`slice(int start, int end)`**, **`map((T value) => U f)`**, **`filter((T value) => bool predicate)`**, **`forEach((T value) => void f)`**, **`sort((T a, T b) => int compare)`**, **`find((T value) => bool predicate)`**. Arrays are fixed-size; for dynamic add/remove/search use **system.List&lt;T&gt;** below.

---

## Core interfaces (built-in)

The following interfaces are part of the language contract and live in the `system` namespace. They are defined in the language specification and must be recognized by the runtime. They are available without explicit import.

| Interface | Defined in | Description |
|-----------|-----------|-------------|
| **Stringable** | [specs.md § Stringable interface](specs.md#stringable-interface) | `string toString() const` — contract for converting reference types to string (used by string concatenation, `(string)` cast, `system.Out.print`). Does not declare `throws`; implementations may throw runtime exceptions if needed. |
| **Cloneable** | [specs.md § Cloneable interface](specs.md#cloneable-interface) | `Self clone()` — contract for object copying (shallow by default). |
| **ValueEquatable** | [specs.md § ValueEquatable interface](specs.md#valueequatable-interface) | `bool valueEquals(const Self|null other)` + `int valueHash()` — contract for structural equality (used by `system.Map` for key lookup). |

---

## system.Out (stdout)

Standard output stream. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `print` | `static void print(string s)` | Writes `s` to standard output without a trailing newline. |
| `println` | `static void println(string s)` | Writes `s` to standard output followed by a newline. |

Overloads of `print` and `println` for other types (e.g. `int`, `float`, `bool`) **must** be provided by the runtime; they behave as if the value were converted to its string representation first.

**Example**

```nl
system.Out.print("Hello");
system.Out.println(" World");
```

---

## system.Err (stderr)

Standard error stream. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `print` | `static void print(string s)` | Writes `s` to standard error without a trailing newline. |
| `println` | `static void println(string s)` | Writes `s` to standard error followed by a newline. |

Same overload rules as `system.Out` apply for other types.

**Example**

```nl
system.Err.println("Error: invalid input");
```

---

## system.In (stdin)

Standard input stream. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `readLine` | `static string|null readLine()` | Reads a line from standard input. Returns `null` on EOF. |

**Example**

```nl
string|null line = system.In.readLine();
```

---

## system.Int

Static parsing and conversion for integer values. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `parse` | `static int parse(string s) throws NumberFormatException` | Parses `s` as a decimal integer. Throws if format is invalid. |
| `tryParse` | `static int\|null tryParse(string s)` | Parses `s` as a decimal integer; returns **`null`** instead of throwing when the format is invalid. Safe-by-default alternative to `parse` for untrusted input (CLI args, files, network data). |
| `toString` | `static string toString(int n)` | Returns the string representation of `n`. Same representation as used for string concatenation and `system.Out.print(n)`. |

**Safety note:** `NumberFormatException` is a *runtime* exception — the compiler does **not** force callers of
`parse` to handle it. A program that feeds untrusted input to `parse` without `try/catch` crashes on malformed
input. Prefer `tryParse` and a `null` check (the naming follows `enum.tryFrom`).

**Example**

```nl
int n = system.Int.parse("42");
string s = system.Int.toString(42);  // "42"

int|null m = system.Int.tryParse(userInput);
if (m == null) {
    system.Err.println("invalid number: " + userInput);
}
```

---

## system.Float

Static parsing and conversion for floating-point values. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `parse` | `static float parse(string s) throws NumberFormatException` | Parses `s` as a float. Throws if format is invalid. |
| `tryParse` | `static float\|null tryParse(string s)` | Parses `s` as a float; returns **`null`** instead of throwing when the format is invalid. Safe-by-default alternative to `parse` for untrusted input. |
| `toString` | `static string toString(float x)` | Returns the string representation of `x`. Same representation as used for string concatenation and `system.Out.print(x)`. |

**Example**

```nl
float x = system.Float.parse("3.14");
string s = system.Float.toString(3.14);  // e.g. "3.14"
float|null y = system.Float.tryParse(userInput);  // null on invalid input
```

---

## system.Bool

Static parsing and conversion for boolean values. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `parse` | `static bool parse(string s) throws IllegalArgumentException` | Parses `s` as a boolean. Accepts exactly `"true"` or `"false"` (case-sensitive). Throws if the string does not match. |
| `tryParse` | `static bool\|null tryParse(string s)` | Same parsing; returns **`null`** instead of throwing when the string matches neither `"true"` nor `"false"`. |
| `toString` | `static string toString(bool b)` | Returns the string representation of `b` (e.g. `"true"` or `"false"`). Same representation as used for string concatenation and `system.Out.print(b)`. |

**Example**

```nl
bool b = system.Bool.parse("true");
string s = system.Bool.toString(true);  // "true"
```

---

## system.String

Static string utilities and **instance methods on string values**. String values (expression type `string`) support the instance methods listed below. The static forms `trim(string)` and `split(string, delimiter)` are also available when passing a string as argument is preferred — e.g. when the string is `const` (parameter, local) and you want to make the read-only intent explicit, or to avoid a temporary receiver.

Conversion of other types to string: primitives use `system.Int.toString`, `system.Float.toString`, and `system.Bool.toString`; reference types that implement the **Stringable** interface (see [specs.md](specs.md)#stringable-interface) are converted by calling `toString()`.

### Instance methods on string

Called on a string value, e.g. `text.length()`, `name.toUpperCase()`.

| Method | Signature | Description |
|--------|------------|-------------|
| `length` | `int length()` | Returns the number of characters in the string. |
| `charAt` | `string charAt(int index)` | Returns the character at `index` as a string of length 1. Throws IndexOutOfBoundsException if out of range. |
| `substring` | `string substring(int start)` | Returns the substring from `start` to the end. |
| `substring` | `string substring(int start, int end)` | Returns the substring from `start` (inclusive) to `end` (exclusive). Throws IndexOutOfBoundsException if indices are invalid. |
| `indexOf` | `int indexOf(string s)` | Returns the index of the first occurrence of `s`, or `-1` if not found. |
| `indexOf` | `int indexOf(string s, int fromIndex)` | Returns the index of the first occurrence of `s` at or after `fromIndex`, or `-1` if not found. |
| `contains` | `bool contains(string s)` | Returns `true` if the string contains `s`. |
| `toUpperCase` | `string toUpperCase()` | Returns a new string with all characters converted to upper case. |
| `toLowerCase` | `string toLowerCase()` | Returns a new string with all characters converted to lower case. |
| `replace` | `string replace(string from, string to)` | Returns a new string with all occurrences of `from` replaced by `to`. |
| `startsWith` | `bool startsWith(string prefix)` | Returns `true` if the string starts with `prefix`. |
| `endsWith` | `bool endsWith(string suffix)` | Returns `true` if the string ends with `suffix`. |
| `trim` | `string trim()` | Returns a new string with leading and trailing whitespace removed. |
| `split` | `string[] split(string delimiter)` | Splits the string on `delimiter` and returns an array of substrings. |

### Static methods (system.String)

Equivalent to the instance methods; useful when the string is `const` or when passing it as argument is preferred.

| Method | Signature | Description |
|--------|------------|-------------|
| `trim` | `static string trim(string s)` | Returns a new string with leading and trailing whitespace removed. Same as `s.trim()`. |
| `split` | `static string[] split(string s, string delimiter)` | Splits `s` on `delimiter` and returns an array of substrings. Same as `s.split(delimiter)`. |

**Example**

```nl
string text = "  Hello, World  ";
int n = text.length();                    // 16
string upper = text.toUpperCase();        // "  HELLO, WORLD  "
string trimmed = text.trim();             // "Hello, World" — or system.String.trim(text)
string[] parts = "a,b,c".split(",");     // or system.String.split("a,b,c", ",")
bool b = text.startsWith("  He");        // true
int i = text.indexOf("World");           // 9
string sub = text.substring(2, 8);       // "Hello,"
```

---

## system.Random

Pseudo-random number generator. Create an instance (optionally with a seed) and call methods to get random values.

**Security:** `system.Random` is a deterministic PRNG intended for games, simulations, and sampling. It is
**not suitable for security purposes** (session tokens, nonces, keys, password reset codes) — its output is
predictable from its seed or from observed values. Use [`system.SecureRandom`](#systemsecurerandom) instead.

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct()` | Creates a generator with an implementation-defined seed. |
| `construct` | `construct(int seed)` | Creates a generator with the given seed (reproducible). |
| `nextInt` | `int nextInt()` | Returns a random integer (full range). |
| `nextInt` | `int nextInt(int bound)` | Returns a random int in `[0, bound)`. |
| `nextFloat` | `float nextFloat()` | Returns a random float in `[0.0, 1.0)`. |

**Example**

```nl
auto rng = new system.Random();
int n = rng.nextInt(100);
float f = rng.nextFloat();
```

---

## system.SecureRandom

Cryptographically secure random number generator (CSPRNG), backed by the operating system's entropy source
(e.g. `/dev/urandom`, `getrandom(2)`, `BCryptGenRandom`). Not seedable — there is no reproducible mode. Use it
for security-sensitive values: session tokens, nonces, keys, salts. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `nextBytes` | `static void nextBytes(byte[] buffer)` | Fills `buffer` entirely with cryptographically secure random bytes. |
| `nextInt` | `static int nextInt()` | Returns a cryptographically secure random integer (full range). |
| `nextInt` | `static int nextInt(int bound)` | Returns a cryptographically secure random int in `[0, bound)`, uniformly distributed (no modulo bias). |

**Example**

```nl
byte[] token = new byte[32];
system.SecureRandom.nextBytes(token);
string tokenB64 = system.text.Encoding.base64Encode(token);
```

---

## system.Uuid

Universally unique identifier generation. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `random` | `static string random()` | Returns a new **UUID version 4** (random) as a lowercase string (e.g. `"550e8400-e29b-41d4-a716-446655440000"`). The 122 random bits **must** come from the CSPRNG (same source as [`system.SecureRandom`](#systemsecurerandom)), making the identifiers unpredictable. |

**Example**

```nl
string id = system.Uuid.random();
```

---

## system.List

Dynamic sequence of elements of type `T`. Use when you need to add or remove elements (unlike fixed-size arrays). Lives in namespace `system`; use as `system.List<T>`.

**Thread safety:** `List<T>` is **not** thread-safe. Heap objects are shared across threads (see [vm.md § Threading model](vm.md#threading-model)). When multiple threads access the same list, the caller must synchronize access using `system.thread.Mutex` (or another synchronization primitive) to avoid data races and corruption.

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct()` | Creates an empty list. |
| `construct` | `construct(T[] initial)` | Creates a list containing the elements of `initial`. |
| `size` | `int size()` | Returns the number of elements. |
| `get` | `T get(int index)` | Returns the element at `index`. Throws IndexOutOfBoundsException if out of range. |
| `set` | `void set(int index, T value)` | Replaces the element at `index`. Throws IndexOutOfBoundsException if out of range. |
| `pushBack` | `void pushBack(T value)` | Appends `value` at the end. |
| `pushFront` | `void pushFront(T value)` | Inserts `value` at the beginning. |
| `popBack` | `T popBack()` | Removes and returns the last element. Throws if list is empty. |
| `popFront` | `T popFront()` | Removes and returns the first element. Throws if list is empty. |
| `add` | `void add(T value)` | Same as pushBack. |
| `remove` | `T remove(int index)` | Removes and returns the element at `index`. Throws IndexOutOfBoundsException if out of range. |
| `contains` | `bool contains(T value)` | Returns `true` if the list contains an element equal to `value`. Equality: primitives and `string` by value; reference types implementing [ValueEquatable](specs.md#valueequatable-interface) by `valueEquals`; otherwise by reference identity. |

Lists support the [for-each loop](specs.md#loops): `for (const auto item : list) { ... }`.

**Example**

```nl
auto list = new system.List<int>();
list.pushBack(1);
list.pushBack(2);
list.pushBack(3);
int n = list.size();       // 3
int first = list.popFront(); // 1
int last = list.popBack();  // 3
list.remove(0);            // remove element at index 0
bool has = list.contains(2); // true if 2 is in the list
```

---

## system.Map

Key-value storage with keys of type `K` and values of type `V`. Lives in namespace `system`; use as `system.Map<K, V>`.

**Thread safety:** `Map<K, V>` is **not** thread-safe. Heap objects are shared across threads (see [vm.md § Threading model](vm.md#threading-model)). When multiple threads access the same map, the caller must synchronize access using `system.thread.Mutex` (or another synchronization primitive) to avoid data races and corruption.

**Key equality semantics:**

- **Primitives** (`int`, `float`, `bool`, `byte`) and **`string`**: keys are compared by value.
- **Reference types implementing [ValueEquatable](specs.md#valueequatable-interface)**: keys are compared using `valueEquals(other)` and `valueHash()` — two objects with the same structure are considered the same key.
- **Other reference types**: keys are compared by reference identity (same object instance). For value-based keys, implement ValueEquatable on the key class.

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct()` | Creates an empty map. |
| `size` | `int size()` | Returns the number of key-value pairs. |
| `get` | `V|null get(K key)` | Returns the value for `key`, or `null` if not present. |
| `set` | `void set(K key, V value)` | Associates `key` with `value` (insert or update). |
| `remove` | `bool remove(K key)` | Removes the entry for `key`. Returns `true` if the key was present. |
| `has` | `bool has(K key)` | Returns `true` if `key` is in the map. |
| `keys` | `K[] keys()` | Returns an array containing all keys in the map. |
| `values` | `V[] values()` | Returns an array containing all values in the map, in the same order as `keys()`. |
| `entries` | `MapEntry<K,V>[] entries()` | Returns an array of all key-value pairs as [MapEntry](#result-types) objects. |
| `forEach` | `void forEach((K key, V value) => void f)` | Invokes `f` for each key-value pair in the map. |

Maps support the [for-each loop](specs.md#loops): `for (const auto entry : map) { ... }` iterates over `MapEntry<K,V>` objects. The iteration order of `keys()`, `values()`, `entries()`, `forEach()`, and for-each is **consistent** (all produce elements in the same order for the same map state), but the specific ordering (e.g. insertion order vs. arbitrary) is **implementation-defined**.

**Example**

```nl
auto map = new system.Map<string, int>();
map.set("one", 1);
map.set("two", 2);
int|null v = map.get("one");  // 1
bool b = map.has("two");      // true
map.remove("one");

// Iteration via keys / values / entries
string[] k = map.keys();              // e.g. ["two"]
int[] vals = map.values();             // e.g. [2]
auto pairs = map.entries();            // MapEntry<string, int>[]

// Callback-based iteration
map.set("three", 3);
map.forEach((string key, int val) => {
    system.Out.println(key + " = " + val);
});

// For-each loop (iterates over MapEntry<K,V>)
for (const auto entry : map) {
    system.Out.println(entry.key + " -> " + entry.value);
}
```

---

## system.io.FileMode

Enum controlling how a file is opened. Used by `system.io.File.open(string path, FileMode mode)`.

| Case | Read | Write | File absent | File exists |
|------|------|-------|-------------|-------------|
| `Read` | ✓ | ✗ | `FileNotFoundException` | Opens at start |
| `Write` | ✗ | ✓ | Creates | Truncates |
| `Append` | ✗ | ✓ | Creates | Opens at end |
| `ReadWrite` | ✓ | ✓ | `FileNotFoundException` | Opens at start |
| `ReadWriteTruncate` | ✓ | ✓ | Creates | Truncates |
| `ReadWriteAppend` | ✓ | ✓ | Creates | Opens at end |

Calling `read`, `readLine` on a handle opened without read capability, or `write`, `flush` on a handle opened without write capability, throws `IOException`.

---

## system.io.File

Static operations on the file system. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `exists` | `static bool exists(string path)` | Returns `true` if a file or directory exists at `path`. |
| `open` | `static FileHandle open(string path) throws FileNotFoundException` | Opens the file at `path` for reading and writing (equivalent to `ReadWrite`). Returns a handle that must be closed. |
| `open` | `static FileHandle open(string path, FileMode mode) throws FileNotFoundException` | Opens the file at `path` with the given mode. Returns a handle that must be closed. |
| `readAllText` | `static string readAllText(string path) throws FileNotFoundException, IOException` | Reads the entire file at `path` as a string. |
| `writeAllText` | `static void writeAllText(string path, string content) throws IOException` | Overwrites the file at `path` with `content`. |
| `glob` | `static string[] glob(string basePath, string pattern) throws IOException` | Returns paths under `basePath` matching `pattern` (glob or regex). See [Glob](#systemiofile-glob). |

**Security (path traversal):** File system APIs (`File.open`, `File.readAllText`, `File.writeAllText`,
`File.glob`, `Directory.create`, `Directory.remove`, `Directory.list`, `Process.setCwd`) perform **no path
sanitization** and operate with the full privileges of the NL process. Absolute paths, `..` traversal, and
symlinks are followed as-is — **path validation is the caller's responsibility**. Never pass user-controlled
input (e.g. from `system.In.readLine()`, `args`, or network data) directly as a path: an attacker could read or
overwrite arbitrary files (`"../../../../etc/shadow"`). For untrusted input, resolve with
[`system.io.Path.normalize`](#systemiopath) and verify the result stays under an allowed base directory (e.g.
`normalized.startsWith(baseDir)`) before any file operation.

**Example**

```nl
if (system.io.File.exists("data.txt")) {
    string content = system.io.File.readAllText("data.txt");
}
auto handle = system.io.File.open("data.txt");
// ... use handle ...
handle.close();

// Open read-only or append
auto ro = system.io.File.open("config.txt", system.io.FileMode.Read);
auto log = system.io.File.open("app.log", system.io.FileMode.Append);
```

#### system.io.File (glob)

Enumerates file system paths under a base directory that match a pattern. The pattern may be **glob** (e.g. `*.txt`, `src/**/*.nl`) or **regex** (regular expression). Matching is applied to the relative path under `basePath`.

| Method | Signature | Description |
|--------|------------|-------------|
| `glob` | `static string[] glob(string basePath, string pattern) throws IOException` | Returns an array of full paths under `basePath` whose relative path matches `pattern`. |

**Example**

```nl
string[] nlFiles = system.io.File.glob("src", ".*\\.nl");
string[] allInLogs = system.io.File.glob("/var/log", ".*");
```

---

## system.io.FileHandle

Represents an open file. Obtained from `system.io.File.open`. Implements resource semantics: the handle should be closed when no longer needed (e.g. via destructor or explicit `close()`).

| Method | Signature | Description |
|--------|------------|-------------|
| `close` | `void close()` | Closes the file and releases the resource. Idempotent. |
| `read` | `int read(byte[] buffer, int offset, int length) throws IOException` | Reads up to `length` bytes into `buffer` at `offset`. Returns the number of bytes read (0 at end of file). |
| `readLine` | `string|null readLine() throws IOException` | Reads one line (up to `\n` or end of file). Returns `null` at end of file. |
| `write` | `void write(byte[] data, int offset, int length) throws IOException` | Writes `length` bytes from `data` starting at `offset`. |
| `write` | `void write(string text) throws IOException` | Writes the string to the file (no trailing `\n` added). |
| `flush` | `void flush() throws IOException` | Flushes the write buffer to the file. |

The runtime may provide a destructor that calls `close()` if the handle goes out of scope without being closed. Multiple calls to `close()` have no effect after the first. **After the handle has been closed, any call to `read`, `readLine`, `write`, or `flush` throws `IOException`.**

**Bounds checking:** `read(buffer, offset, length)` and `write(data, offset, length)` **must** throw
`IndexOutOfBoundsException` (before performing any I/O) when `offset < 0`, `length < 0`, or
`offset + length > buffer.length()` — the check must be immune to integer overflow of `offset + length`. The same
rule applies to `TcpStream.read`/`write` and `UdpSocket` buffers.

**Example**

```nl
auto handle = system.io.File.open("data.txt");
try {
    string|null line;
    while ((line = handle.readLine()) != null) {
        system.Out.println(line);
    }
} finally {
    handle.close();
}
```

---

## system.io.Directory

Directory operations. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `list` | `static string[] list(string path) throws IOException` | Returns the names of entries (files and directories) in the directory at `path`. |
| `create` | `static void create(string path) throws IOException` | Creates the directory at `path` (and parent directories if needed). |
| `remove` | `static void remove(string path) throws IOException` | Removes the empty directory at `path`. |
| `exists` | `static bool exists(string path)` | Returns `true` if a directory exists at `path`. |

**Example**

```nl
string[] entries = system.io.Directory.list("src");
system.io.Directory.create("out/gen");
system.io.Directory.remove("tmp");
```

---

## system.io.Path

Path manipulation. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `join` | `static string join(string[] segments)` | Joins path segments with the platform separator. |
| `dirname` | `static string dirname(string path)` | Returns the directory part of `path`. |
| `basename` | `static string basename(string path)` | Returns the last segment of `path`. |
| `extension` | `static string|null extension(string path)` | Returns the file extension (e.g. `".nl"`) or `null` if none. |
| `normalize` | `static string normalize(string path)` | Returns a normalized path (resolves `.`, `..`, redundant separators). |

**Example**

```nl
string path = system.io.Path.join(new string[]{"src", "com", "example", "App.nl"});
string dir = system.io.Path.dirname(path);
string base = system.io.Path.basename(path);
string|null ext = system.io.Path.extension(path);
```

---

## system.io.Grep

Searches for lines matching a pattern (regex) in a file or under a directory. Returns match information (path, line number, line content).

| Method | Signature | Description |
|--------|------------|-------------|
| `search` | `static GrepMatch[] search(string pattern, string path) throws IOException` | Returns all lines in the file at `path` that match `pattern` (regex). |
| `search` | `static GrepMatch[] search(string pattern, string dirPath, bool recursive) throws IOException` | If `recursive` is `true`, searches all files under `dirPath`; otherwise only the file or directory at `dirPath`. Returns matches with path, line number, and line. |

**GrepMatch** (result type): has `string path`, `int lineNumber`, `string line`.

**Example**

```nl
auto matches = system.io.Grep.search("error|Error", "app.log");
for (auto m : matches) {
    system.Out.println(m.path + ":" + m.lineNumber + " " + m.line);
}
auto all = system.io.Grep.search("TODO", "src", true);
```

---

## system.net.TcpListener

Listens for incoming TCP connections on a host and port. Create a listener, then call `accept()` to block until a client connects; the returned `TcpStream` is used for reading and writing.

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct(string host, int port) throws IOException` | Binds the listener to `host` and `port`. |
| `accept` | `TcpStream accept() throws IOException` | Blocks until a client connects; returns a stream for the new connection. |
| `close` | `void close()` | Stops listening and releases the port. Idempotent. |

**Example**

```nl
auto listener = new system.net.TcpListener("0.0.0.0", 8080);
while (true) {
    auto stream = listener.accept();
    // handle stream (e.g. in another task)
    stream.close();
}
listener.close();
```

---

## system.net.TcpStream

Represents a connected TCP stream, obtained from `TcpListener.accept()` or `TcpStream.connect()`. Used for reading and writing bytes; must be closed when done.

| Method | Signature | Description |
|--------|------------|-------------|
| `connect` | `static TcpStream connect(string host, int port) throws IOException` | Connects to `host:port` and returns a stream. |
| `read` | `int read(byte[] buffer, int offset, int length) throws IOException` | Reads up to `length` bytes into `buffer` at `offset`. Returns number of bytes read, or 0 on EOF. |
| `write` | `void write(byte[] data, int offset, int length) throws IOException` | Writes `length` bytes from `data` at `offset` to the stream. |
| `close` | `void close()` | Closes the connection. Idempotent. |

The runtime may provide a destructor that calls `close()` if the stream goes out of scope without being closed. **After the stream has been closed, any call to `read` or `write` throws `IOException`.**

**Bounds checking:** `read` and `write` follow the same rule as [FileHandle](#systemiofilehandle): `offset < 0`,
`length < 0`, or `offset + length > buffer.length()` throws `IndexOutOfBoundsException` before any I/O.

**Example**

```nl
auto stream = system.net.TcpStream.connect("127.0.0.1", 8080);
byte[] buf = new byte[256];
int n = stream.read(buf, 0, buf.length);
stream.write(buf, 0, n);
stream.close();
```

---

## system.net.UdpSocket

UDP datagram socket for connectionless communication. Can send and receive datagram packets to/from any address.

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct()` | Creates an unbound UDP socket. |
| `bind` | `void bind(string host, int port) throws IOException` | Binds the socket to `host:port` for receiving. |
| `send` | `void send(string host, int port, byte[] data) throws IOException` | Sends `data` to `host:port`. |
| `receive` | `int receive(byte[] buffer) throws IOException` | Receives a datagram into `buffer`. Returns number of bytes received. |
| `close` | `void close()` | Closes the socket. Idempotent. |

After the socket has been closed, any call to `send` or `receive` throws `IOException`. `receive(buffer)` fills
the buffer from index `0` and never writes past `buffer.length()`; datagrams larger than the buffer are truncated
(the excess bytes are discarded).

**Example**

```nl
auto socket = new system.net.UdpSocket();
socket.bind("0.0.0.0", 9999);
byte[] buf = new byte[512];
int n = socket.receive(buf);
socket.send("127.0.0.1", 9998, buf);
socket.close();
```

---

## system.net.Http

Simple HTTP client for GET and POST requests. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `get` | `static HttpResponse get(string url) throws IOException` | Performs a GET request; returns status code and body. |
| `post` | `static HttpResponse post(string url, string body) throws IOException` | Performs a POST request with the given body. |

**HttpResponse** (result type): has `int statusCode`, `string body`, and optionally `string[] headers`. Encoding (e.g. UTF-8) is implementation-defined.

**TLS (https):** For `https://` URLs, implementations **must validate the server certificate by default**: chain
of trust against the platform trust store, expiration, and hostname verification. A failed validation **must**
throw `IOException` — silently proceeding would expose users to man-in-the-middle attacks. No option to disable
validation is specified; custom trust stores, certificate pinning, and a TLS wrapper for `TcpStream` may be
specified in a future version.

**Example**

```nl
auto res = system.net.Http.get("https://example.com/");
system.Out.println(res.statusCode + " " + res.body);
```

---

## system.thread.Thread

Represents a thread of execution. The thread is created with a task (anonymous function with no parameters) and is started explicitly; the caller can wait for it to finish with `join()`.

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct(() => void task)` | Creates a thread that will run `task` when started. |
| `start` | `void start()` | Starts the thread. The task runs in parallel. Must not be called more than once. |
| `join` | `void join() throws InterruptedException` | Blocks until the thread has finished. |
| `join` | `bool join(int timeoutMillis) throws InterruptedException` | Blocks until the thread has finished or `timeoutMillis` have elapsed. Returns `true` if the thread finished, `false` if timeout. |
| `isAlive` | `bool isAlive()` | Returns `true` if the thread has been started and has not yet finished; `false` otherwise. Non-blocking. |
| `interrupt` | `void interrupt()` | Requests interruption of this thread: sets its interrupt flag. Non-blocking, callable from any thread. |
| `isInterrupted` | `bool isInterrupted()` | Returns `true` if this thread's interrupt flag is set (i.e. an `interrupt()` is pending and has not been delivered). Does not clear the flag. |
| `sleep` | `static void sleep(int millis) throws InterruptedException` | Puts the current thread to sleep for `millis` milliseconds. |

**Interruption.** `interrupt()` sets a per-thread flag; it never stops a thread by force. The flag is turned into an `InterruptedException` only at the three *interruption points* — `join()`, `join(int)` and `sleep(int)` — which test it before blocking and while blocked:

* A thread blocked in an interruption point when `interrupt()` is called wakes up and throws `InterruptedException` promptly, instead of waiting for the thread or the timeout it was waiting for. `join(int)` reports an interruption by throwing, never by returning `false`.
* Throwing **clears** the flag: after delivery, `isInterrupted()` is `false` again and a subsequent wait blocks normally unless the thread is interrupted anew.
* The flag is **sticky** otherwise. Interrupting a thread that is not currently at an interruption point (including one that has not been started yet, or has already finished) is well-defined: nothing is lost, the flag stays set — observable through `isInterrupted()` — and the next interruption point the thread reaches throws immediately. A task that never reaches one simply runs to completion.
* Everything else that can block is **not** an interruption point: `Mutex.lock()`, `Semaphore.acquire()` and the blocking `system.io`/`system.net` operations declare no `InterruptedException` and are unaffected by the flag.
* There is no `Thread` object for the thread running `main`, so nothing can interrupt it; `sleep()`/`join()` called from `main` never throw `InterruptedException` (the declaration is still part of their signature, and callers must still handle or declare it).

**Example**

```nl
auto t = new system.thread.Thread(() => {
    system.thread.Thread.sleep(100);
    system.Out.println("done");
});
t.start();
t.join();
```

**Example (interruption)**

```nl
auto worker = new system.thread.Thread(() => {
    try {
        system.thread.Thread.sleep(60000);
        system.Out.println("finished waiting");
    } catch (InterruptedException e) {
        system.Out.println("cancelled");
    }
});
worker.start();
worker.interrupt();   // "cancelled", without waiting a minute
worker.join();
```

---

## system.thread.Mutex

Mutual exclusion lock for protecting shared data across threads. Use `lock()` before accessing shared state and `unlock()` when done (or use a scope guard if the language provides one).

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct()` | Creates an unlocked mutex. |
| `lock` | `void lock()` | Acquires the lock; blocks if another thread holds it. |
| `unlock` | `void unlock()` | Releases the lock. Must be called by the thread that holds it. |
| `tryLock` | `bool tryLock()` | Tries to acquire the lock without blocking. Returns `true` if acquired, `false` otherwise. |

**Example**

```nl
auto mutex = new system.thread.Mutex();
mutex.lock();
// ... access shared state ...
mutex.unlock();
```

---

## system.thread.Semaphore

Counting semaphore for limiting concurrency or signaling between threads. Holds a non-negative count; `acquire()` decrements it (blocking if zero) and `release()` increments it.

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct(int initialCount)` | Creates a semaphore with the given initial count. |
| `acquire` | `void acquire()` | Decrements the count; blocks if count is zero. |
| `release` | `void release()` | Increments the count, possibly unblocking a waiting thread. |
| `tryAcquire` | `bool tryAcquire()` | Tries to decrement without blocking. Returns `true` if successful. |

**Example**

```nl
auto sem = new system.thread.Semaphore(2);
sem.acquire();
// ... do work ...
sem.release();
```

---

## system.time.DateTime

Represents a date and time, optionally with a timezone. All methods are **static** for factory/parsing; instance methods for accessors and formatting.

| Method | Signature | Description |
|--------|------------|-------------|
| `now` | `static DateTime now()` | Current date and time in the default timezone. |
| `now` | `static DateTime now(TimeZone zone)` | Current date and time in the given timezone. |
| `parse` | `static DateTime parse(string s) throws FormatException` | Parses an ISO-8601-like string (e.g. `2025-03-01T14:30:00Z` or `2025-03-01T15:30:00+01:00`). |
| `getYear` | `int getYear()` | Year (e.g. 2025). |
| `getMonth` | `int getMonth()` | Month 1–12. |
| `getDay` | `int getDay()` | Day of month 1–31. |
| `getHour` | `int getHour()` | Hour 0–23. |
| `getMinute` | `int getMinute()` | Minute 0–59. |
| `getSecond` | `int getSecond()` | Second 0–59. |
| `getTimeZone` | `TimeZone getTimeZone()` | The timezone of this date/time. |
| `withTimeZone` | `DateTime withTimeZone(TimeZone zone)` | Returns a new DateTime representing the same instant in `zone`. |
| `toUtc` | `DateTime toUtc()` | Same instant in UTC. |
| `format` | `string format(string pattern)` | Formats using placeholders (e.g. `yyyy-MM-dd HH:mm`). |

**Example**

```nl
auto now = system.time.DateTime.now();
system.Out.println(now.format("yyyy-MM-dd HH:mm"));
auto utc = now.toUtc();
auto paris = system.time.TimeZone.get("Europe/Paris");
auto inParis = now.withTimeZone(paris);
```

---

## system.time.TimeZone

Represents a timezone (e.g. for converting and formatting dates). All methods are **static** except where noted.

| Method | Signature | Description |
|--------|------------|-------------|
| `getDefault` | `static TimeZone getDefault()` | The process default timezone. |
| `get` | `static TimeZone get(string id) throws IllegalArgumentException` | Timezone by ID (e.g. `"Europe/Paris"`, `"UTC"`). |
| `getId` | `string getId()` | The timezone identifier. |
| `getOffsetMinutes` | `int getOffsetMinutes(DateTime at)` | Offset from UTC in minutes at the given instant. |

**Example**

```nl
auto tz = system.time.TimeZone.get("America/New_York");
auto dt = system.time.DateTime.now(tz);
```

---

## system.Env

Environment variables of the current process. All methods are **static**.

**Thread safety:** `Env.set()` and `Env.remove()` modify the process environment, which is a shared global
resource; calling them concurrently from multiple threads (or while another thread reads with `Env.get()` /
`Env.list()`) is **undefined behavior** on most platforms. Synchronize with `system.thread.Mutex`, or set all
variables from the main thread before spawning threads.

**Security:** environment variables frequently contain secrets (API keys, connection strings). Do not display
the output of `Env.list()`/`Env.get()` to untrusted users. Variables like `PATH` or `LD_PRELOAD` influence how
subprocesses resolve executables and libraries — setting them from untrusted input then calling
`system.ps.Process.run` enables code injection.

| Method | Signature | Description |
|--------|------------|-------------|
| `get` | `static string|null get(string name)` | Returns the value of the variable `name`, or `null` if unset. |
| `set` | `static void set(string name, string value)` | Sets the variable `name` to `value` for the current process and its children. |
| `remove` | `static void remove(string name)` | Removes the variable `name` from the current process environment. |
| `list` | `static string[] list()` | Returns the names of all environment variables. |

**Example**

```nl
string|null home = system.Env.get("HOME");
system.Env.set("MY_VAR", "value");
string[] names = system.Env.list();
```

---

## system.ps

Process listing, subprocess execution, current process identity, working directory, and exit. The namespace exposes:

- **system.ps.Process** — class with **static** methods for listing processes, running subprocesses, and querying/changing the current process (pid, cwd, exit).
- **system.ps.ProcessInfo** — result type returned by `Process.list()` (one process entry).
- **system.ps.ProcessResult** — result type returned by `Process.run()` (exit code, stdout, stderr).

### system.ps.Process

All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `list` | `static ProcessInfo[] list()` | Returns information for all processes visible to the current process (process list). Platform-dependent; see note below. |
| `list` | `static ProcessInfo[] list(int pid)` | Returns information for the process with the given PID, or an empty array if not found. Platform-dependent; see note below. |
| `run` | `static ProcessResult run(string[] args) throws IOException` | Starts a process with the given arguments; waits for it to finish. Returns exit code and captured stdout/stderr. |
| `run` | `static ProcessResult run(string command) throws IOException` | Runs the command via the platform shell; waits for completion. |
| `pid` | `static int pid()` | Returns the current process ID. |
| `exit` | `static void exit(int code)` | Terminates the process with the given exit code. **Terminal statement**: does not return; the compiler treats subsequent code as unreachable (same as `throw`). |
| `getCwd` | `static string getCwd()` | Returns the current working directory path. |
| `setCwd` | `static void setCwd(string path) throws IOException` | Changes the current working directory. |

**ProcessInfo** (result type): has `int pid`, `string command`, `string[] args`, `string|null user` (or platform-specific fields). See [Result types](#result-types) above.

**ProcessResult** (result type): has `int exitCode`, `string stdout`, `string stderr`. See [Result types](#result-types) above.

**Platform support for `list`/`list(pid)`:** process listing is inherently OS-specific (no portable standard API). Implementations are expected to support it on Linux and macOS at minimum; on any platform where it isn't implemented, `list()` returns an empty array and `list(pid)` behaves as "not found" rather than throwing. `command`/`args` splitting is best-effort on platforms that reconstruct the command line from a single string (no exact quoting round-trip guaranteed).

**Security (command injection):** `run(string command)` passes the string to the platform shell for interpretation. Never interpolate user-controlled input (e.g. from `system.In.readLine()`, `args`, or network data) into the command string — an attacker could inject arbitrary shell commands. Prefer `run(string[] args)`, which bypasses the shell and passes arguments directly to the executable.

**Example**

```nl
auto processes = system.ps.Process.list();
for (auto p : processes) {
    system.Out.println(p.pid + " " + p.command);
}
auto one = system.ps.Process.list(1234);

auto result = system.ps.Process.run("ls -la");
system.Out.println(result.stdout);
// Safe alternative when using user input: run(string[] args) bypasses the shell
// auto result = system.ps.Process.run(["grep", pattern, "/etc/passwd"]);
int myPid = system.ps.Process.pid();
string cwd = system.ps.Process.getCwd();
system.ps.Process.setCwd("/tmp");
system.ps.Process.exit(1);
```

---

## system.text.Regex

Regular expression match and replace. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `match` | `static bool match(string pattern, string input)` | Returns `true` if `pattern` is found **anywhere** in `input` (**partial match**, like `grep` and PHP's `preg_match`). For a full match, anchor the pattern explicitly: `"^…$"`. |
| `matchFirst` | `static RegexMatch|null matchFirst(string pattern, string input)` | Returns the first match with groups, or `null`. Partial-match semantics, like `match`. |
| `replace` | `static string replace(string pattern, string input, string replacement)` | Replaces all matches of `pattern` in `input` with `replacement`. |
| `split` | `static string[] split(string pattern, string input)` | Splits `input` by occurrences of `pattern`. |
| `escape` | `static string escape(string s)` | Returns `s` with every regex metacharacter escaped, so the result matches `s` **literally**. Always use it when embedding user-controlled input in a pattern. |

**RegexMatch** (result type): has `string fullMatch`, `string[] groups` (capture groups). Exact API is implementation-defined.

**Example**

```nl
bool ok = system.text.Regex.match("\\d+", "abc123");        // true — partial match
bool full = system.text.Regex.match("^\\d+$", "abc123");    // false — anchored (full match)
string cleaned = system.text.Regex.replace("\\s+", "a  b  c", " ");
string[] parts = system.text.Regex.split("\\s*,\\s*", "a, b , c");

// Embedding user input in a pattern: escape it
string safe = system.text.Regex.escape(userInput);           // "a.b*" -> "a\\.b\\*"
bool found = system.text.Regex.match(safe, haystack);
```

---

## system.text.Encoding

String and byte conversion, base64. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `encodeUtf8` | `static byte[] encodeUtf8(string s)` | Encodes the string to UTF-8 bytes. |
| `decodeUtf8` | `static string decodeUtf8(byte[] bytes)` | Decodes UTF-8 bytes to a string. |
| `base64Encode` | `static string base64Encode(byte[] bytes)` | Encodes bytes to a base64 string. |
| `base64Decode` | `static byte[] base64Decode(string s) throws FormatException` | Decodes a base64 string to bytes. |

**Example**

```nl
byte[] raw = system.text.Encoding.encodeUtf8("hello");
string back = system.text.Encoding.decodeUtf8(raw);
string b64 = system.text.Encoding.base64Encode(raw);
byte[] decoded = system.text.Encoding.base64Decode(b64);
```

---

## system.text.json

JSON ([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)) parsing and serialization. A JSON document is represented as a **`JsonValue`** tree; **`system.text.json.Json`** provides the static `parse`/`tryParse`/`stringify` entry points. All types live in the `system.text.json` namespace.

### system.text.json.JsonValue

Abstract base class for the six JSON value kinds. Implements [Stringable](specs.md#stringable-interface): `toString()` returns the compact JSON serialization of the value (and its descendants).

| Method | Signature | Description |
|--------|------------|-------------|
| `toString` | `string toString() const` | Compact JSON serialization of this value. |
| `isNull` | `bool isNull() const` | `true` if this is `JsonNull`. |
| `isBool` | `bool isBool() const` | `true` if this is `JsonBool`. |
| `isNumber` | `bool isNumber() const` | `true` if this is `JsonNumber`. |
| `isString` | `bool isString() const` | `true` if this is `JsonString`. |
| `isArray` | `bool isArray() const` | `true` if this is `JsonArray`. |
| `isObject` | `bool isObject() const` | `true` if this is `JsonObject`. |
| `asBool` | `bool asBool() const` | Returns the boolean value. Throws `InvalidCastException` if this is not `JsonBool`. |
| `asNumber` | `float asNumber() const` | Returns the numeric value. Throws `InvalidCastException` if this is not `JsonNumber`. |
| `asString` | `string asString() const` | Returns the string value. Throws `InvalidCastException` if this is not `JsonString`. |
| `asArray` | `JsonArray asArray() const` | Returns this value as `JsonArray`. Throws `InvalidCastException` if this is not `JsonArray`. |
| `asObject` | `JsonObject asObject() const` | Returns this value as `JsonObject`. Throws `InvalidCastException` if this is not `JsonObject`. |

**Concrete subclasses:**

| Class | Fields / constructor | Description |
|-------|------------------------|-------------|
| `JsonNull` | `construct()` | Represents JSON `null`. No fields. |
| `JsonBool` | `construct(bool value)`, `public bool value` | Wraps a boolean. |
| `JsonNumber` | `construct(float value)`, `public float value` | Wraps a JSON number. JSON has a single numeric type; NL represents it as `float` — integers beyond 2⁵³ lose precision (same trade-off as JavaScript's `JSON.parse`). |
| `JsonString` | `construct(string value)`, `public string value` | Wraps a string. |
| `JsonArray` | see below | Ordered list of `JsonValue`. |
| `JsonObject` | see below | Ordered key-value map of `string` to `JsonValue`. |

All six are `final class readonly`: immutable after construction (a `JsonArray`/`JsonObject` cannot be reassigned to a different underlying list/map, but its contents can still be mutated through `add`/`set`/`remove`, like any `readonly` property holding a mutable object).

### system.text.json.JsonArray

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct()` | Creates an empty array. |
| `length` | `int length() const` | Returns the number of elements. |
| `get` | `JsonValue get(int index) const` | Returns the element at `index`. Throws `IndexOutOfBoundsException` if out of range. |
| `set` | `void set(int index, JsonValue value)` | Replaces the element at `index`. Throws `IndexOutOfBoundsException` if out of range. |
| `add` | `void add(JsonValue value)` | Appends `value` to the end. |
| `values` | `JsonValue[] values() const` | Returns a snapshot array of all elements, in order. |

### system.text.json.JsonObject

| Method | Signature | Description |
|--------|------------|-------------|
| `construct` | `construct()` | Creates an empty object. |
| `size` | `int size() const` | Returns the number of key-value pairs. |
| `get` | `JsonValue\|null get(string key) const` | Returns the value for `key`, or `null` if the key is **absent**. |
| `set` | `void set(string key, JsonValue value)` | Associates `key` with `value` (insert or update). |
| `has` | `bool has(string key) const` | Returns `true` if `key` is present. |
| `remove` | `bool remove(string key)` | Removes the entry for `key`. Returns `true` if it was present. |
| `keys` | `string[] keys() const` | Returns all keys, in the same order as `entries()`. |
| `entries` | `system.MapEntry<string, JsonValue>[] entries() const` | Returns all key-value pairs as [MapEntry](#result-types) objects. |

**Absent key vs. JSON `null`:** `get(key)` returns NL `null` only when `key` is not in the object. A key present with the JSON value `null` returns a `JsonNull` instance (for which `isNull()` is `true`), not NL `null`. This mirrors the same distinction `system.Map.get` makes between "key absent" and "value is null" for reference-typed maps.

Key order follows insertion order (same iteration-order guarantee as [system.Map](#systemmap)).

### system.text.json.Json

Parsing and serialization entry points. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `parse` | `static JsonValue parse(string text) throws JsonFormatException` | Parses `text` as JSON and returns the resulting value tree. Throws `JsonFormatException` on malformed input. |
| `tryParse` | `static JsonValue\|null tryParse(string text)` | Same parsing; returns `null` instead of throwing when `text` is not valid JSON. Safe-by-default alternative to `parse` for untrusted input. |
| `stringify` | `static string stringify(JsonValue value)` | Serializes `value` to a compact JSON string (no extra whitespace). |
| `stringify` | `static string stringify(JsonValue value, int indent)` | Serializes `value` to a pretty-printed JSON string, indenting nested levels by `indent` spaces. |

**Example**

```nl
// Parsing
system.text.json.JsonValue root = system.text.json.Json.parse(
    "{\"name\": \"Ada\", \"tags\": [\"eng\", \"math\"], \"active\": true}"
);

system.text.json.JsonValue|null nameValue = root.asObject().get("name");
string name = nameValue != null ? nameValue.asString() : "unknown"; // "Ada"

system.text.json.JsonValue|null tagsValue = root.asObject().get("tags");
if (tagsValue != null) {
    system.text.json.JsonArray tags = tagsValue.asArray();
    for (const auto tag : tags.values()) {
        system.Out.println(tag.asString()); // "eng", then "math"
    }
}

// Safe parsing of untrusted input
system.text.json.JsonValue|null parsed = system.text.json.Json.tryParse(userInput);
if (parsed == null) {
    system.Out.println("invalid JSON");
}

// Building and serializing
auto obj = new system.text.json.JsonObject();
obj.set("name", new system.text.json.JsonString("Ada"));
obj.set("active", new system.text.json.JsonBool(true));
string compact = system.text.json.Json.stringify(obj);     // {"name":"Ada","active":true}
string pretty = system.text.json.Json.stringify(obj, 2);   // indented, 2 spaces per level

// Precise error reporting
try {
    system.text.json.Json.parse("{ \"a\": }");
}
catch (system.text.json.JsonFormatException ex) {
    system.Out.println("JSON error at line " + ex.line + ", column " + ex.column +
        ": expected " + ex.expectedToken + " but found " + ex.foundToken);
}
```

---

## system.db

SQL database connectivity. The `system.db` namespace defines driver-agnostic types (`Connection`, `PreparedStatement`, `ResultSet`, `Row`, `ColumnType`) used by every driver; concrete drivers live in sub-namespaces (`system.db.sqlite`, `system.db.mysql`) and expose a static factory that returns a `Connection`. From the caller's point of view, code written against `Connection` is portable across drivers.

**Placeholders.** All prepared statements use positional **`?`** placeholders (SQL92 style). Named placeholders (`:name`, `$1`) are not specified. Binding uses **0-based** indices to align with NL array and list conventions.

**Security (SQL injection).** **Never** build SQL by concatenating user-controlled input (from `system.In.readLine()`, `args`, network data, HTTP bodies, JSON values, etc.) into the query text — any single quote, semicolon, or comment in the input can rewrite the statement (`"' OR 1=1 --"`). Always use `Connection.prepare(...)` and pass user data through `bindX(...)` — bound parameters are transmitted out-of-band and can never be interpreted as SQL. `Connection.query(string)` and `Connection.execute(string)` accept only the SQL text (no user data merged in) and are intended for constant, hard-coded statements (`"BEGIN"`, `"CREATE TABLE ..."`, admin scripts); passing an interpolated string to them is the same defect as `Process.run(string)` on user input (see [system.ps.Process](#systempsprocess)).

**Thread safety.** `Connection`, `PreparedStatement`, and `ResultSet` are **not thread-safe** — a `Connection` may only be used from one thread at a time (including all statements and result sets derived from it). To share a database from multiple threads, either open one `Connection` per thread or serialize access with `system.thread.Mutex`. Heap objects are shared across threads (see [vm.md § Threading model](vm.md#threading-model)); driver internals rely on that guarantee but do not add locking.

**Resource lifetime.** `Connection`, `PreparedStatement`, and `ResultSet` hold native handles and **must** be closed (explicit `close()` or destructor) — otherwise file descriptors, server-side cursors, and network connections leak. Closing a `Connection` also closes every `PreparedStatement` and `ResultSet` derived from it. `close()` is idempotent on all three types. **After a connection/statement/result set has been closed, any subsequent operation (other than `close()` itself) throws `SqlException`**, mirroring the closed-handle rule of [FileHandle](#systemiofilehandle) and [TcpStream](#systemnettcpstream).

### system.db.ColumnType

Enum describing the SQL type of a column value, as reported by `Row.columnType(int)`. All drivers map their native types to this small set.

| Case | Description |
|------|-------------|
| `Integer` | Signed integer (mapped to NL `int`). |
| `Float` | Floating-point number (mapped to NL `float`). |
| `Text` | Text string (mapped to NL `string`, decoded as UTF-8). |
| `Blob` | Binary data (mapped to NL `byte[]`). |
| `Bool` | Boolean value (mapped to NL `bool`). Drivers without a native boolean type (e.g. SQLite) report boolean columns as `Integer` unless the schema declares them otherwise. |
| `Null` | SQL `NULL`. |

### system.db.Connection

Represents an open database connection. Instances are obtained from a driver factory (`system.db.sqlite.Sqlite.open`, `system.db.mysql.Mysql.connect`) and are **opaque** to user code — the concrete subclass is an implementation detail of the driver. All methods are instance methods.

| Method | Signature | Description |
|--------|------------|-------------|
| `prepare` | `PreparedStatement prepare(string sql) throws SqlException` | Compiles `sql` into a reusable prepared statement. The statement's placeholder count is derived from the `?` markers in `sql`. |
| `query` | `ResultSet query(string sql) throws SqlException` | Executes a **constant** SQL query with no parameters and returns the result set. Convenience for hard-coded statements — do **not** pass user input; use `prepare(...)` + `bindX(...)` instead. |
| `execute` | `int execute(string sql) throws SqlException` | Executes a **constant** SQL statement with no parameters (typically `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, `BEGIN`, `COMMIT`, `ROLLBACK`). Returns the number of rows affected, or `0` for statements that do not affect rows. Same "no user input" caveat as `query(string)`. |
| `beginTransaction` | `void beginTransaction() throws SqlException` | Starts a new transaction. Throws `SqlException` if a transaction is already active on this connection. |
| `commit` | `void commit() throws SqlException` | Commits the current transaction. Throws `SqlException` if no transaction is active. |
| `rollback` | `void rollback() throws SqlException` | Rolls back the current transaction. Throws `SqlException` if no transaction is active. |
| `lastInsertId` | `int\|null lastInsertId()` | Returns the row ID generated by the most recent `INSERT` on this connection (auto-increment / rowid / `LAST_INSERT_ID()`), or `null` if no such row exists (no insert has been performed, or the last statement did not generate an ID). |
| `isClosed` | `bool isClosed()` | Returns `true` if `close()` has been called on this connection. |
| `close` | `void close()` | Closes the connection and releases all associated resources (open statements, result sets, pending transaction is implicitly rolled back). Idempotent. |

The runtime may provide a destructor that calls `close()` if a `Connection` goes out of scope without being closed. Multiple calls to `close()` have no effect after the first. **After the connection has been closed, `prepare`, `query`, `execute`, `beginTransaction`, `commit`, `rollback`, and `lastInsertId` throw `SqlException`.**

### system.db.PreparedStatement

Represents a compiled SQL statement with `?` placeholders. Obtained from `Connection.prepare(string)`. Bind values to placeholders (0-based) with `bindX(...)`, then run the statement with `query()` (for `SELECT`) or `execute()` (for `INSERT`/`UPDATE`/`DELETE`/DDL). A statement can be reused: call `reset()` (or bind again) to run it with new parameters.

| Method | Signature | Description |
|--------|------------|-------------|
| `parameterCount` | `int parameterCount()` | Returns the number of `?` placeholders in the statement. |
| `bindInt` | `void bindInt(int index, int value) throws SqlException` | Binds `value` to placeholder at `index` (0-based). |
| `bindFloat` | `void bindFloat(int index, float value) throws SqlException` | Binds `value` to placeholder at `index`. |
| `bindBool` | `void bindBool(int index, bool value) throws SqlException` | Binds `value` to placeholder at `index`. Drivers without a native boolean type encode `true`/`false` as `1`/`0`. |
| `bindString` | `void bindString(int index, string value) throws SqlException` | Binds `value` to placeholder at `index` (transmitted as UTF-8 text). |
| `bindBytes` | `void bindBytes(int index, byte[] value) throws SqlException` | Binds `value` to placeholder at `index` (transmitted as `BLOB`/`VARBINARY`). |
| `bindNull` | `void bindNull(int index) throws SqlException` | Binds SQL `NULL` to placeholder at `index`. |
| `query` | `ResultSet query() throws SqlException` | Executes the statement with the currently bound parameters and returns a result set. Use for `SELECT`. |
| `execute` | `int execute() throws SqlException` | Executes the statement with the currently bound parameters and returns the number of rows affected. Use for `INSERT`/`UPDATE`/`DELETE`/DDL. |
| `reset` | `void reset() throws SqlException` | Clears all bound parameters and resets the statement so it can be re-executed with new bindings. Does not close the statement. |
| `isClosed` | `bool isClosed()` | Returns `true` if `close()` has been called on this statement (or if the parent connection has been closed). |
| `close` | `void close()` | Closes the statement and releases its compiled form. Idempotent. |

Binding methods throw `SqlException` when `index < 0` or `index >= parameterCount()`, when the driver reports a type-conversion failure, or when the statement (or its connection) has been closed. `query`/`execute` throw `SqlException` if any placeholder is left unbound, if the SQL execution fails (constraint violation, syntax error reported lazily, deadlock, connection lost, etc.), or if the statement (or its connection) has been closed.

### system.db.ResultSet

Represents the rows produced by a `SELECT`. Rows are consumed one at a time by calling `next()`, which returns the next [Row](#systemdbrow) or `null` when the set is exhausted. A `ResultSet` holds a native cursor and **must** be closed when done.

| Method | Signature | Description |
|--------|------------|-------------|
| `next` | `Row\|null next() throws SqlException` | Advances the cursor and returns the next row, or `null` when there are no more rows. |
| `columnCount` | `int columnCount()` | Returns the number of columns in each row of the result set. |
| `columnName` | `string columnName(int index) throws SqlException` | Returns the name of the column at `index` (0-based). Throws `SqlException` if `index` is out of range. |
| `isClosed` | `bool isClosed()` | Returns `true` if `close()` has been called on this result set (or if the parent statement/connection has been closed). |
| `close` | `void close()` | Closes the result set and releases the underlying cursor. Idempotent. |

`ResultSet` supports the [for-each loop](specs.md#loops): `for (const auto row : results) { ... }` iterates by repeatedly calling `next()` until `null` is returned. The result set is not automatically closed at the end of the loop — call `close()` (or rely on the destructor). Any `Row` returned by `next()` is invalidated on the next call to `next()` or `close()`; the caller must copy any value they need to keep past that point.

**After a result set has been closed**, `next`, `columnCount` (when the count was not cached before close), and `columnName` throw `SqlException`.

### system.db.Row

Represents a single row from a [ResultSet](#systemdbresultset). Columns are accessed by 0-based index; typed accessors return the value converted to the requested NL type. Every typed accessor returns a nullable union (`T|null`) — SQL `NULL` maps to NL `null`.

| Method | Signature | Description |
|--------|------------|-------------|
| `columnCount` | `int columnCount() const` | Returns the number of columns in the row (same value as `ResultSet.columnCount()`). |
| `columnType` | `ColumnType columnType(int index) const` | Returns the [ColumnType](#systemdbcolumntype) of the value at `index`. Throws `IndexOutOfBoundsException` if `index` is out of range. |
| `isNull` | `bool isNull(int index) const` | Returns `true` if the value at `index` is SQL `NULL`. Throws `IndexOutOfBoundsException` if `index` is out of range. |
| `getInt` | `int\|null getInt(int index) const` | Returns the value at `index` as `int`, or `null` if the value is SQL `NULL`. Throws `InvalidCastException` if the value cannot be represented as an `int` (e.g. text that does not parse, blob). |
| `getFloat` | `float\|null getFloat(int index) const` | Returns the value at `index` as `float`, or `null` if SQL `NULL`. Throws `InvalidCastException` on type mismatch. |
| `getBool` | `bool\|null getBool(int index) const` | Returns the value at `index` as `bool`, or `null` if SQL `NULL`. Throws `InvalidCastException` on type mismatch. |
| `getString` | `string\|null getString(int index) const` | Returns the value at `index` as `string` (UTF-8-decoded for text columns), or `null` if SQL `NULL`. Throws `InvalidCastException` on type mismatch. |
| `getBytes` | `byte[]\|null getBytes(int index) const` | Returns the value at `index` as a byte array, or `null` if SQL `NULL`. Throws `InvalidCastException` on type mismatch. |

**Column lookup by name.** For readability, every typed accessor has an overload that takes the column name instead of the index: `getInt(string columnName)`, `getFloat(string columnName)`, `getBool(string columnName)`, `getString(string columnName)`, `getBytes(string columnName)`, `isNull(string columnName)`, `columnType(string columnName)`. These overloads resolve the name to the corresponding index (case-sensitive, first match) and throw `SqlException` if no column with that name exists in the row. When a query aliases columns (`SELECT id AS user_id ...`), the alias is the name to use.

`Row` instances are valid only until the next call to `ResultSet.next()` or `ResultSet.close()`. Passing a stale `Row` reference to any accessor throws `SqlException`.

### Example (driver-agnostic)

```nl
// Open a connection (driver-specific factory)
system.db.Connection db = system.db.sqlite.Sqlite.open("app.db");
try {
    // DDL and constant SQL: query()/execute() with no user input
    db.execute("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT, age INTEGER)");

    // Parameterized INSERT — user input is bound, never interpolated
    auto ins = db.prepare("INSERT INTO users(name, age) VALUES (?, ?)");
    try {
        db.beginTransaction();
        ins.bindString(0, "Ada");
        ins.bindInt(1, 36);
        ins.execute();

        ins.reset();
        ins.bindString(0, "Linus");
        ins.bindInt(1, 55);
        ins.execute();

        db.commit();
    }
    catch (system.db.SqlException ex) {
        db.rollback();
        throw ex;
    }
    finally {
        ins.close();
    }

    // Parameterized SELECT — for-each iteration on the ResultSet
    auto q = db.prepare("SELECT id, name, age FROM users WHERE age >= ? ORDER BY id");
    q.bindInt(0, 18);
    auto rows = q.query();
    try {
        for (const auto row : rows) {
            int|null id = row.getInt("id");
            string|null name = row.getString("name");
            int|null age = row.getInt("age");
            system.Out.println(id + " " + name + " " + age);
        }
    }
    finally {
        rows.close();
        q.close();
    }
}
finally {
    db.close();
}
```

---

## system.db.sqlite

SQLite driver. Provides a static factory that returns a `system.db.Connection` backed by an embedded SQLite database. All methods are **static**.

### system.db.sqlite.SqliteOpenMode

Enum controlling how a SQLite database file is opened.

| Case | File absent | File exists |
|------|-------------|-------------|
| `ReadOnly` | `SqlException` | Opens read-only |
| `ReadWrite` | `SqlException` | Opens read-write |
| `ReadWriteCreate` | Creates | Opens read-write |

### system.db.sqlite.Sqlite

| Method | Signature | Description |
|--------|------------|-------------|
| `open` | `static Connection open(string path) throws SqlException` | Opens the SQLite database at `path` in `ReadWriteCreate` mode. Returns a `system.db.Connection`. |
| `open` | `static Connection open(string path, SqliteOpenMode mode) throws SqlException` | Opens the SQLite database at `path` with the given mode. |
| `openMemory` | `static Connection openMemory() throws SqlException` | Opens a private, in-memory database. The database exists only for the lifetime of the returned `Connection`. |

**Security (path traversal).** `open(string path, ...)` performs no path sanitization — the same rule as [system.io.File](#systemiofile) applies. Never pass user-controlled input directly as the database path; resolve with [`system.io.Path.normalize`](#systemiopath) and verify the result stays under an allowed base directory. Even in `ReadOnly` mode, opening an arbitrary file interprets it as a SQLite database (which may cause reads of unexpected content and leak information via error messages).

**Example**

```nl
system.db.Connection db = system.db.sqlite.Sqlite.open(
    "cache.db",
    system.db.sqlite.SqliteOpenMode.ReadWriteCreate
);
// ... use db ...
db.close();

// In-memory database (unit tests, ephemeral caches)
auto mem = system.db.sqlite.Sqlite.openMemory();
mem.execute("CREATE TABLE t(x INTEGER)");
mem.close();
```

---

## system.db.mysql

MySQL / MariaDB driver. Provides a static factory that returns a `system.db.Connection` backed by a TCP connection to a MySQL-compatible server.

### system.db.mysql.MysqlConfig

Configuration for a MySQL connection. Instantiated by user code and passed to `Mysql.connect(...)`. All fields are `public`; the constructor takes them positionally, but callers should prefer [named parameters](specs.md#named-parameters) for readability.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `host` | `string` | — | Server hostname or IP address. |
| `port` | `int` | `3306` | Server TCP port. |
| `user` | `string` | — | Login user name. |
| `password` | `string` | — | Login password (empty string when none). |
| `database` | `string` | `""` | Initial database to select (empty string means "no default database"; must then be selected with `USE ...`). |
| `useTls` | `bool` | `true` | If `true`, negotiate TLS with the server after the initial handshake and validate the server certificate. |

Constructor: `construct(string host, int port, string user, string password, string database, bool useTls)`.

### system.db.mysql.Mysql

| Method | Signature | Description |
|--------|------------|-------------|
| `connect` | `static Connection connect(MysqlConfig config) throws SqlException` | Opens a connection to the MySQL server described by `config`. Returns a `system.db.Connection`. |

**TLS (`useTls = true`).** When TLS is requested, the implementation **must validate the server certificate by default**: chain of trust against the platform trust store, expiration, and hostname verification (against `config.host`). A failed validation **must** throw `SqlException` — silently proceeding would expose credentials and query data to man-in-the-middle attacks. Same contract as [`system.net.Http`](#systemnethttp) for `https://` URLs. No option to disable validation is specified; custom trust stores and certificate pinning may be specified in a future version.

**Security (credentials).** Never log or print `config.password`, and never include it in error messages or in the string form of a `MysqlConfig`. Prefer loading credentials from `system.Env` or a secrets file rather than embedding them in source. Do not interpolate `config.host`, `config.user`, or `config.database` from untrusted input without validation — a hostile hostname can point at an attacker-controlled server that harvests credentials.

**Example**

```nl
auto config = new system.db.mysql.MysqlConfig(
    host: "db.example.com",
    port: 3306,
    user: system.Env.get("DB_USER") ?? "app",
    password: system.Env.get("DB_PASSWORD") ?? "",
    database: "app",
    useTls: true
);
system.db.Connection db = system.db.mysql.Mysql.connect(config);
try {
    auto q = db.prepare("SELECT COUNT(*) FROM orders WHERE customer_id = ?");
    q.bindInt(0, customerId);
    auto rows = q.query();
    try {
        auto row = rows.next();
        if (row != null) {
            int|null count = row.getInt(0);
            system.Out.println("orders: " + (count ?? 0));
        }
    }
    finally {
        rows.close();
        q.close();
    }
}
finally {
    db.close();
}
```

---

## Exceptions

Standard exceptions used by the system API. The hierarchy (Runtime vs Checked) is defined in the language specification (see [specs.md](specs.md)#exception-class-hierarchy). **Runtime** exceptions do not require a `throws` declaration; **Checked** exceptions must be declared or handled.

| Exception | Kind | Namespace | Thrown by |
|-----------|------|-----------|-----------|
| `IndexOutOfBoundsException` | Runtime | `system` | Array access via `[]` when index is out of range; `system.List.get`, `system.List.set`, `system.List.remove` when index is out of range; `system.List.popBack`, `system.List.popFront` when the list is empty; **`string.charAt`, `string.substring`** when index or range is out of range; **byte-array `read`/`write` bounds violations** on `FileHandle`, `TcpStream` (negative `offset`/`length`, or `offset + length > buffer.length()`) |
| `NumberFormatException` | Runtime | `system` | `system.Int.parse`, `system.Float.parse` when the string format is invalid |
| `IllegalArgumentException` | Runtime | `system` | `enum.from()` when value does not match any case; `system.Bool.parse` when string is not `"true"` or `"false"`; `system.time.TimeZone.get()` when the timezone ID is unknown |
| `StackOverflowException` | Runtime | `system` | Thrown by the VM when the call stack is exhausted (e.g. infinite recursion). Resource-exhaustion signal, not a control-flow mechanism: the maximum depth is implementation-defined, and a runtime applying tail call optimization may never throw it for recursion in tail position — see [optimizations.md](optimizations.md#tail-call-optimization-and-stackoverflowexception) |
| `FileNotFoundException` | Checked | `system.io` | `system.io.File.open`, `system.io.File.readAllText` when the path does not exist or is not a file (modes `Read`, `ReadWrite` require existing file) |
| `IOException` | Checked | `system.io` | `system.io.File`, `system.io.Directory`, `system.io.Path`, `system.io.Grep`, and other I/O failures; **read/write/flush on closed FileHandle** |
| `IOException` | Checked | `system.net` | `system.net.TcpListener`, `system.net.TcpStream`, `system.net.UdpSocket`, `system.net.Http` on connection or read/write failure; **read/write on closed TcpStream; send/receive on closed UdpSocket** |
| `IOException` | Checked | `system.ps` | `system.ps.Process.run()`, `system.ps.Process.setCwd()` when the process cannot be started or the path is invalid |
| `InterruptedException` | Checked | `system.thread` | Thrown by `join()`/`join(int)`/`sleep(int)` when the waiting thread's interrupt flag is set — see [Interruption](#systemthreadthread). |
| `FormatException` | Checked | `system.time` | `system.time.DateTime.parse()` when the string format is invalid. |
| `FormatException` | Checked | `system.text` | `system.text.Encoding.base64Decode()` when the string is not valid base64. |
| `JsonFormatException` | Checked | `system.text.json` | `system.text.json.Json.parse()` when `text` is not valid JSON. Extends `system.text.FormatException`; carries `line`, `column`, `expectedToken`, `foundToken`. |
| `SqlException` | Checked | `system.db` | `system.db.Connection`, `system.db.PreparedStatement`, `system.db.ResultSet`, and driver factories (`system.db.sqlite.Sqlite.open`, `system.db.mysql.Mysql.connect`) on any database failure: connection error, TLS validation failure, SQL syntax error, constraint violation, deadlock, type-conversion failure during `bindX`, out-of-range placeholder or column name, use of a closed connection/statement/result set, use of a stale `Row`. Carries `sqlState` (SQLSTATE code, empty string when the driver does not provide one) and `errorCode` (driver-specific numeric code, `0` when not provided). |

---

## Notes

- The exact signatures (e.g. overloads for `print` on `Out`/`Err` for other types, or buffering behavior) are left to the language specification and runtime documentation.
