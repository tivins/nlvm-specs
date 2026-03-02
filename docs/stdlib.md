# NL Standard Library (System API)

This document describes the virtual standard library provided by the NL runtime for system interaction: standard streams (out, err, in), parsing, file system access (including glob, directories, paths), network sockets (TCP, UDP) and HTTP, threads and synchronization (Mutex, Semaphore), date/time with timezone, grep-style text search, environment variables, process listing and subprocess execution, string and regex utilities, random and UUID, encoding and base64. All types live in the **`system`**, **`system.io`**, **`system.net`**, **`system.thread`**, **`system.time`**, **`system.Env`**, **`system.ps`**, and **`system.text`** namespaces and are available without explicit import in user code (built-in bindings). Return types written as **`T|null`** (or other union types like **`Type1|Type2|null`**) denote nullable or union types as defined in the language specification (see [specs.md](specs.md)#union-types-and-explicit-nullable).

---

## Summary

* [Namespaces](#namespaces)
* [Result types](#result-types)
* [Arrays (built-in)](#arrays-built-in)
* [system.Out (stdout)](#systemout-stdout)
* [system.Err (stderr)](#systemerr-stderr)
* [system.In (stdin)](#systemin-stdin)
* [system.Int](#systemint)
* [system.Float](#systemfloat)
* [system.Bool](#systembool)
* [system.String](#systemstring)
* [system.Random](#systemrandom)
* [system.Uuid](#systemuuid)
* [system.List](#systemlist)
* [system.Map](#systemmap)
* [system.Env](#systemenv)
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
* [Exceptions](#exceptions)

---

## Namespaces

| Namespace   | Purpose                          |
|------------|-----------------------------------|
| `system`   | Standard streams (Out, Err, In), parsing and conversion (Int, Float, Bool), String, Random, Uuid, **List&lt;T&gt;**, **Map&lt;K,V&gt;**, **MapEntry&lt;K,V&gt;** |
| `system.io`| File system (File, FileHandle, Directory, Path), glob, Grep |
| `system.net`| Network (TcpListener, TcpStream, UdpSocket, Http) |
| `system.thread`| Threads (Thread), synchronization (Mutex, Semaphore) |
| `system.time`| Date, time, timezone (DateTime, TimeZone) |
| `system.ps`| Process listing (Process.list, ProcessInfo), subprocess execution and current process (Process.run, Process.pid, Process.getCwd, Process.setCwd, Process.exit) |
| `system.text`| Regex, Encoding (UTF-8, base64) |

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

Array types `T[]` are part of the language (see [specs.md](specs.md)#arrays). Creation: `new T[]{ ... }` (initializer list) or `new T[n]` (fixed size). Access by index: `arr[i]` (get/set); out-of-range access throws **IndexOutOfBoundsException**. Built-in methods: **`length()`**, **`slice(int start, int end)`**, **`map((T value) => U f)`**, **`filter((T value) => bool predicate)`**, **`forEach((T value) => void f)`**, **`sort((T a, T b) => int compare)`**, **`find((T value) => bool predicate)`**. Arrays are fixed-size; for dynamic add/remove use **system.List&lt;T&gt;** below.

---

## system.Out (stdout)

Standard output stream. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `print` | `static void print(string s)` | Writes `s` to standard output without a trailing newline. |
| `println` | `static void println(string s)` | Writes `s` to standard output followed by a newline. |

Overloads of `print` and `println` for other types (e.g. `int`, `float`, `bool`) may be provided by the runtime; they behave as if the value were converted to its string representation first.

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
| `parseInt` | `static int parseInt(string s) throws NumberFormatException` | Parses `s` as a decimal integer. Throws if format is invalid. |
| `toString` | `static string toString(int n)` | Returns the string representation of `n`. Same representation as used for string concatenation and `system.Out.print(n)`. |

**Example**

```nl
int n = system.Int.parseInt("42");
string s = system.Int.toString(42);  // "42"
```

---

## system.Float

Static parsing and conversion for floating-point values. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `parseFloat` | `static float parseFloat(string s) throws NumberFormatException` | Parses `s` as a float. Throws if format is invalid. |
| `toString` | `static string toString(float x)` | Returns the string representation of `x`. Same representation as used for string concatenation and `system.Out.print(x)`. |

**Example**

```nl
float x = system.Float.parseFloat("3.14");
string s = system.Float.toString(3.14);  // e.g. "3.14"
```

---

## system.Bool

Static conversion for boolean values. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `toString` | `static string toString(bool b)` | Returns the string representation of `b` (e.g. `"true"` or `"false"`). Same representation as used for string concatenation and `system.Out.print(b)`. |

**Example**

```nl
string s = system.Bool.toString(true);  // "true"
```

---

## system.String

Static string utilities and **instance methods on string values**. String values (expression type `string`) support the instance methods listed below; `system.String` also provides static methods.

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

### Static methods (system.String)

| Method | Signature | Description |
|--------|------------|-------------|
| `trim` | `static string trim(string s)` | Returns a new string with leading and trailing whitespace removed. |
| `split` | `static string[] split(string s, string delimiter)` | Splits `s` on `delimiter` and returns an array of substrings. |

**Example**

```nl
string text = "  Hello, World  ";
int n = text.length();                    // 15
string upper = text.toUpperCase();        // "  HELLO, WORLD  "
string trimmed = system.String.trim(text); // "Hello, World"
string[] parts = system.String.split("a,b,c", ",");
bool b = text.startsWith("  He");        // true
int i = text.indexOf("World");           // 9
string sub = text.substring(2, 8);        // "Hello"
```

---

## system.Random

Pseudo-random number generator. Create an instance (optionally with a seed) and call methods to get random values.

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

## system.Uuid

Universally unique identifier generation. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `random` | `static string random()` | Returns a new UUID as a string (e.g. `"550e8400-e29b-41d4-a716-446655440000"`). |

**Example**

```nl
string id = system.Uuid.random();
```

---

## system.List

Dynamic sequence of elements of type `T`. Use when you need to add or remove elements (unlike fixed-size arrays). Lives in namespace `system`; use as `system.List<T>`.

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
```

---

## system.Map

Key-value storage with keys of type `K` and values of type `V`. Lives in namespace `system`; use as `system.Map<K, V>`.

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

## system.io.File

Static operations on the file system. All methods are **static**.

| Method | Signature | Description |
|--------|------------|-------------|
| `exists` | `static bool exists(string path)` | Returns `true` if a file or directory exists at `path`. |
| `open` | `static FileHandle open(string path) throws FileNotFoundException` | Opens the file at `path` for reading and/or writing. Returns a handle that must be closed. |
| `readAllText` | `static string readAllText(string path) throws FileNotFoundException, IOException` | Reads the entire file at `path` as a string. |
| `writeAllText` | `static void writeAllText(string path, string content) throws IOException` | Overwrites the file at `path` with `content`. |
| `glob` | `static string[] glob(string basePath, string pattern) throws IOException` | Returns paths under `basePath` matching `pattern` (glob or regex). See [Glob](#systemiofile-glob). |

**Example**

```nl
if (system.io.File.exists("data.txt")) {
    string content = system.io.File.readAllText("data.txt");
}
auto handle = system.io.File.open("data.txt");
// ... use handle ...
handle.close();
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

The runtime may provide a destructor that calls `close()` if the handle goes out of scope without being closed. Multiple calls to `close()` have no effect after the first.

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
string path = system.io.Path.join(new string[] {"src", "com", "example", "App.nl"});
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

The runtime may provide a destructor that calls `close()` if the stream goes out of scope without being closed.

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
| `sleep` | `static void sleep(int millis) throws InterruptedException` | Puts the current thread to sleep for `millis` milliseconds. |

**Example**

```nl
auto t = new system.thread.Thread(() => {
    system.thread.Thread.sleep(100);
    system.Out.println("done");
});
t.start();
t.join();
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
| `list` | `static ProcessInfo[] list()` | Returns information for all processes visible to the current process (process list). |
| `list` | `static ProcessInfo[] list(int pid)` | Returns information for the process with the given PID, or an empty array if not found. |
| `run` | `static ProcessResult run(string[] args) throws IOException` | Starts a process with the given arguments; waits for it to finish. Returns exit code and captured stdout/stderr. |
| `run` | `static ProcessResult run(string command) throws IOException` | Runs the command via the platform shell; waits for completion. |
| `pid` | `static int pid()` | Returns the current process ID. |
| `exit` | `static void exit(int code)` | Terminates the process with the given exit code. Does not return. |
| `getCwd` | `static string getCwd()` | Returns the current working directory path. |
| `setCwd` | `static void setCwd(string path) throws IOException` | Changes the current working directory. |

**ProcessInfo** (result type): has `int pid`, `string command`, `string[] args`, `string|null user` (or platform-specific fields). See [Result types](#result-types) above.

**ProcessResult** (result type): has `int exitCode`, `string stdout`, `string stderr`. See [Result types](#result-types) above.

**Example**

```nl
auto processes = system.ps.Process.list();
for (auto p : processes) {
    system.Out.println(p.pid + " " + p.command);
}
auto one = system.ps.Process.list(1234);

auto result = system.ps.Process.run("ls -la");
system.Out.println(result.stdout);
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
| `match` | `static bool match(string pattern, string input)` | Returns `true` if `input` matches `pattern`. |
| `matchFirst` | `static RegexMatch|null matchFirst(string pattern, string input)` | Returns the first match with groups, or `null`. |
| `replace` | `static string replace(string pattern, string input, string replacement)` | Replaces all matches of `pattern` in `input` with `replacement`. |
| `split` | `static string[] split(string pattern, string input)` | Splits `input` by occurrences of `pattern`. |

**RegexMatch** (result type): has `string fullMatch`, `string[] groups` (capture groups). Exact API is implementation-defined.

**Example**

```nl
bool ok = system.text.Regex.match("\\d+", "123");
string cleaned = system.text.Regex.replace("\\s+", "a  b  c", " ");
string[] parts = system.text.Regex.split("\\s*,\\s*", "a, b , c");
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

## Exceptions

Standard exceptions used by the system API. The hierarchy (Runtime vs Checked) is defined in the language specification (see [specs.md](specs.md)#exception-class-hierarchy). **Runtime** exceptions do not require a `throws` declaration; **Checked** exceptions must be declared or handled.

| Exception | Kind | Namespace | Thrown by |
|-----------|------|-----------|-----------|
| `IndexOutOfBoundsException` | Runtime | `system` | Array access via `[]` when index is out of range; `system.List.get`, `system.List.set` when index is out of range; `system.List.popBack`, `system.List.popFront` when the list is empty; **`string.charAt`, `string.substring`** when index or range is out of range |
| `NumberFormatException` | Runtime | `system` | `system.Int.parseInt`, `system.Float.parseFloat` when the string format is invalid |
| `IllegalArgumentException` | Runtime | `system.time` | `system.time.TimeZone.get()` when the timezone ID is unknown |
| `FileNotFoundException` | Checked | `system.io` | `system.io.File.open`, `system.io.File.readAllText` when the path does not exist or is not a file |
| `IOException` | Checked | `system.io` | `system.io.File`, `system.io.Directory`, `system.io.Path`, `system.io.Grep`, and other I/O failures |
| `IOException` | Checked | `system.net` | `system.net.TcpListener`, `system.net.TcpStream`, `system.net.UdpSocket`, `system.net.Http` on connection or read/write failure |
| `IOException` | Checked | `system.ps` | `system.ps.Process.run()`, `system.ps.Process.setCwd()` when the process cannot be started or the path is invalid |
| `InterruptedException` | Checked | `system.thread` | Thrown when a thread is interrupted while blocked in `join()` or `sleep()`. |
| `FormatException` | Checked | `system.time` | `system.time.DateTime.parse()` when the string format is invalid. |
| `FormatException` | Checked | `system.text` | `system.text.Encoding.base64Decode()` when the string is not valid base64. |

---

## Notes

- The exact signatures (e.g. overloads for `print` on `Out`/`Err` for other types, or buffering behavior) are left to the language specification and runtime documentation.
