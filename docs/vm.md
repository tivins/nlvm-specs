# NL Virtual Machine Specification

This document describes the **execution model**, **bytecode format**, and **instruction set** of the NL virtual
machine (VM). It complements [specs.md](specs.md) (language semantics), [compiler.md](compiler.md) (compile-time
checks), and [stdlib.md](stdlib.md) (standard library). Those documents define *what* the language allows and *what
the compiler verifies*; this document defines *how* compiled code is represented and executed at runtime.

## Summary

* [Architecture overview](#architecture-overview)
* [Value representation](#value-representation)
* [Module format](#module-format)
    * [Format version](#format-version)
    * [Optimization level](#optimization-level)
    * [Constant pool](#constant-pool)
    * [Type descriptors](#type-descriptors)
    * [Module integrity](#module-integrity)
    * [Class descriptor](#class-descriptor)
    * [Field descriptor](#field-descriptor)
    * [Method descriptor](#method-descriptor)
* [Runtime structures](#runtime-structures)
    * [Heap](#heap)
    * [Call frame and operand stack](#call-frame-and-operand-stack)
    * [Static storage](#static-storage)
* [Object model](#object-model)
    * [Object layout](#object-layout)
    * [Array layout](#array-layout)
    * [String representation](#string-representation)
    * [Enum representation](#enum-representation)
* [Instruction set](#instruction-set)
    * [Notation](#notation)
    * [Stack manipulation](#stack-manipulation)
    * [Constants](#constants)
    * [Local variable access](#local-variable-access)
    * [Integer arithmetic](#integer-arithmetic)
    * [Float arithmetic](#float-arithmetic)
    * [Type conversions](#type-conversions)
    * [Comparison](#comparison)
    * [Control flow](#control-flow)
    * [Object operations](#object-operations)
    * [Array operations](#array-operations)
    * [Field access](#field-access)
    * [Method invocation](#method-invocation)
    * [String operations](#string-operations)
    * [Exception operations](#exception-operations)
    * [Miscellaneous](#miscellaneous)
* [Method dispatch](#method-dispatch)
    * [Static methods](#static-methods)
    * [Instance methods](#instance-methods)
    * [Constructors](#constructors)
    * [Super calls](#super-calls)
    * [Interface dispatch](#interface-dispatch)
* [Exception handling](#exception-handling)
    * [Exception table](#exception-table)
    * [Throw and stack unwinding](#throw-and-stack-unwinding)
    * [Finally blocks](#finally-blocks)
    * [Stack trace construction](#stack-trace-construction)
* [Closures and anonymous functions](#closures-and-anonymous-functions)
* [Compilation strategies](#compilation-strategies)
    * [Templates (monomorphization)](#templates-monomorphization)
    * [Ref parameters (boxing)](#ref-parameters-boxing)
    * [String concatenation](#string-concatenation)
    * [Union types at runtime](#union-types-at-runtime)
    * [For-each loops](#for-each-loops)
    * [Named and optional parameters](#named-and-optional-parameters)
    * [Operator overloading](#operator-overloading)
    * [Increment and decrement operators](#increment-and-decrement-operators)
    * [Nullish coalescing and elvis operators](#nullish-coalescing-and-elvis-operators)
    * [Switch statements](#switch-statements)
    * [Match expressions](#match-expressions)
* [Program startup](#program-startup)
* [VM invocation (nlvm)](#vm-invocation-nlvm)
* [Standard library binding](#standard-library-binding)
* [Threading model](#threading-model)
* [Garbage collection contract](#garbage-collection-contract)
* [Object lifecycle](#object-lifecycle)

For optimization-related guarantees (string interning, devirtualization, JIT, etc.), see [optimizations.md](optimizations.md).

---

## Architecture overview

The NL VM is a **stack-based virtual machine**. The compiler (front-end) produces *modules* — one per source file —
containing bytecode, metadata, and a constant pool. The VM loads modules, resolves inter-module references, and
executes bytecode instructions that operate on an operand stack.

Key properties:

- **One module per source file.** Each `.nl` source file compiles to one module. A module contains exactly one
  top-level class or enum definition (as required by [specs.md § Source code files](specs.md#source-code-files)).
- **Statically typed bytecode.** The compiler resolves all types, overloads, and template instantiations before
  emitting bytecode. The VM does not perform type inference.
- **Typed opcodes for arithmetic; tagged values on the stack.** Each stack slot carries a type tag alongside its
  data. Arithmetic opcodes are type-specific (e.g. `IADD` for integers, `FADD` for floats) so the VM can
  fast-path without inspecting tags. Tags serve as a safety net and are required for generic operations
  (comparisons, `TO_STRING`, null checks).

---

## Value representation

Every value manipulated by the VM — on the operand stack, in local variables, in object fields, and in array
elements — is a **tagged value**: a type tag followed by a fixed-size data payload.

### Primitive type sizes

| Type    | Tag | Size    | Range / Format |
|---------|-----|---------|----------------|
| `int`   | `1` | 64 bits | Signed two's complement (−2⁶³ to 2⁶³−1). |
| `float` | `2` | 64 bits | IEEE 754 double-precision. |
| `bool`  | `3` | 8 bits  | `0` = false, `1` = true. |
| `byte`  | `4` | 8 bits  | Unsigned (0 to 255). |

### Reference types

| Kind      | Tag | Payload |
|-----------|-----|---------|
| `null`    | `0` | None (zero payload). |
| Reference | `5` | Pointer to a heap object (object, array, string, or closure). |

Strings are heap-allocated, immutable reference objects. A string value on the stack is a reference (tag `5`)
pointing to a string object.

### Stack slot size

Each stack slot is 9 bytes: 1 byte tag + 8 bytes data. Smaller types (`bool`, `byte`) are stored in the low-order
byte(s) of the 8-byte data field; unused bytes are zero-padded. Implementations may use a wider slot or a different
internal layout as long as the observable semantics are preserved.

---

## Module format

A compiled module contains the following sections in order. Multi-byte integers are **big-endian**.

| Offset | Field | Type | Description |
|--------|-------|------|-------------|
| 0 | `magic` | `u32` | `0x4E4C4D00` (ASCII `NLM\0`). |
| 4 | `version` | `u16` | Module format version (currently `3`; version 2 added the line-number table and the integrity trailer, version 3 adds `opt_level`). See [Format version](#format-version). |
| 6 | `opt_level` | `u8` | Optimization level applied when this module was compiled. Version 3 and later only. See [Optimization level](#optimization-level). |
| 7 | `constant_pool_count` | `u16` | Number of entries in the constant pool (1-indexed; entry 0 is unused). |
| 9 | `constant_pool` | varies | Constant pool entries (see below). |
| … | `this_class` | `u16` | Constant pool index of a `CLASS` entry for the class defined in this module. |
| … | `class_flags` | `u16` | Bit flags for the class (see below). |
| … | `super_class` | `u16` | Constant pool index of a `CLASS` entry for the parent class, or `0` if none. |
| … | `interfaces_count` | `u16` | Number of implemented interfaces. |
| … | `interfaces` | `u16[]` | Constant pool indices of `CLASS` entries for each interface. |
| … | `fields_count` | `u16` | Number of fields. |
| … | `fields` | varies | Field descriptors (see below). |
| … | `methods_count` | `u16` | Number of methods. |
| … | `methods` | varies | Method descriptors (see below). |
| … | `hash_algo` | `u8` | Integrity trailer: hash algorithm (`0` = none, `1` = SHA-256). See [Module integrity](#module-integrity). |
| … | `hash` | `u8[]` | Present only if `hash_algo ≠ 0`. Hash of all preceding bytes of the module (32 bytes for SHA-256). |

Class flag bits:

| Bit | Name | Description |
|-----|------|-------------|
| 0 | `READONLY` | All instance properties are immutable after construction (`class readonly`). |
| 1 | `INTERFACE` | This module defines an interface, not a class. |
| 2 | `ENUM` | This module defines an enum. |
| 3 | `ABSTRACT` | Abstract class (`abstract class`). The VM **must** reject `NEW` targeting a class with this flag (verification error at link time; if reached at runtime, the VM aborts execution with an error). |
| 4 | `FINAL` | Final class (`final class`). At link time, the VM **must** reject any module whose `super_class` refers to a class with this flag. |

`ABSTRACT` and `FINAL` are mutually exclusive (the compiler enforces E049; the loader rejects a module with both bits set). For a module with the `INTERFACE` flag, the `interfaces` list holds the **extended** interfaces (`interface A extends B, C`; see [specs.md § Interface inheritance](specs.md#interface-inheritance)) and `super_class` is `0`.

### Format version

`version` is a **shared namespace owned by this specification**. It is split as follows:

| Range | Owner | Meaning |
|-------|-------|---------|
| `0x0000`–`0x7FFF` | This specification | Spec-defined layouts. `1`, `2` and `3` are assigned; the rest are unassigned and **must not** be used by an implementation. |
| `0x8000`–`0xFFFF` | Implementations | Implementation-defined layouts. A conforming VM **may** reject any of them; no portable meaning is attached. |

Rules:

- An implementation that needs a layout the specification does not define **must** take a number from the
  implementation range, never an unassigned spec number. Two implementations may then pick the same number, but
  neither collides with a future spec version.
- A VM **must** reject a module whose `version` it does not support (load error, non-zero exit); it **must not**
  guess the layout from the magic number or the integrity trailer, both of which are identical across versions.
- A VM that supports version 3 **should** also accept versions 1 and 2, which remain readable: version 3 is
  version 2 with `opt_level` inserted after `version`, and every following field is unchanged.

### Optimization level

`opt_level` records the optimization level the compiler applied, i.e. the `-O<n>` it was invoked with (see
[compiler.md § Options](compiler.md#options)). Its purpose is to make an artifact self-describing: given a `.nlm`
on disk, a human or the VM can tell which build it is looking at.

| Value | Meaning |
|-------|---------|
| `0` | No optimization applied. Corresponds to `nlc -O0`. |
| `1` | An implementation-chosen set of the optimizations in [optimizations.md § Compiler optimizations](optimizations.md#compiler-optimizations). Corresponds to `nlc -O1`. |
| `2`–`255` | Implementation-defined higher levels. |

Rules:

- The compiler **must** write the level it actually applied. Emitting `0` while applying optimizations is
  non-conforming.
- `opt_level` is **metadata, not a directive**: it does not change how the VM executes the module. A VM **may**
  reject a module whose `opt_level` it does not recognize, but **must not** silently clamp or reinterpret it.
- A program built from modules with **different** `opt_level` values is valid and **must not** be rejected: the
  standard library and third-party modules are routinely distributed prebuilt at a level the current build did not
  choose. Reporting the mix is a diagnostic concern (see [VM invocation](#vm-invocation-nlvm)).
- In a version 1 or 2 module the level is **unrecorded**, not `0`. Such a module may well have been optimized; a VM
  **must not** report it as unoptimized.

### Constant pool

Each constant pool entry starts with a 1-byte tag, followed by tag-specific data.

| Tag | Name | Data | Description |
|-----|------|------|-------------|
| `1` | `INT` | `i64` | Integer literal. |
| `2` | `FLOAT` | `f64` | Float literal (IEEE 754). |
| `3` | `STRING` | `u16` length + UTF-8 bytes | String literal or identifier. |
| `4` | `CLASS` | `u16` name_index | Fully qualified class name (index to `STRING` entry). |
| `5` | `FIELD_REF` | `u16` class_index + `u16` name_index + `u16` type_index | Reference to a field. |
| `6` | `METHOD_REF` | `u16` class_index + `u16` name_index + `u16` descriptor_index | Reference to a method. |
| `7` | `TYPE_DESC` | `u16` desc_index | Type descriptor string (index to `STRING` entry). |

### Type descriptors

Type descriptors are strings stored in the constant pool. They encode types in a human-readable format:

| Type | Descriptor string |
|------|-------------------|
| `int` | `"int"` |
| `float` | `"float"` |
| `bool` | `"bool"` |
| `byte` | `"byte"` |
| `string` | `"string"` |
| `void` | `"void"` |
| Class reference | Fully qualified name, e.g. `"com.example.MyClass"` |
| Array | Element type + `"[]"`, e.g. `"int[]"`, `"com.example.MyClass[]"` |
| Union | Types joined by `"\|"`, e.g. `"string\|null"`, `"int\|string\|null"` |
| Null alone | `"null"` |
| Method | `"(" params ") -> " return_type`, e.g. `"(int, string) -> void"` |

### Module integrity

Every module ends with an **integrity trailer**: a `hash_algo` byte followed, when `hash_algo ≠ 0`, by the hash
of **all preceding bytes** of the module (from the magic number up to and including the last method descriptor).

| `hash_algo` | Algorithm | Hash size |
|-------------|-----------|-----------|
| `0` | None (no hash follows) | 0 bytes |
| `1` | SHA-256 | 32 bytes |

Rules:

- Compilers **should** emit a SHA-256 hash by default (`hash_algo = 1`).
- When a hash is present, the VM **must** recompute and verify it at load time; on mismatch, the module is
  rejected and the VM refuses to run (load error, non-zero exit).
- `hash_algo = 0` is allowed (e.g. for fast development builds); loading unhashed modules is permitted but an
  implementation may provide a strict mode that rejects them.

**Trust model.** The hash detects **corruption and accidental tampering**; it does not authenticate the module's
origin (an attacker who can replace the module can recompute the hash). `.nlm` files must be treated as trusted
code, with the same care as native executables: the directories on the module path (`--module-path`) must not be
writable by untrusted users. Digital signatures (origin authentication) are out of scope for this version and may
be specified later.

### Class descriptor

The `this_class` index points to a `CLASS` constant pool entry whose name is the fully qualified class name
(e.g. `"com.example.myProject.MyClass"`). The `super_class` index is `0` if the class does not extend another class.

### Field descriptor

Each field is encoded as:

| Field | Type | Description |
|-------|------|-------------|
| `flags` | `u16` | Bit flags (see below). |
| `name_index` | `u16` | Constant pool `STRING` index for the field name. |
| `type_index` | `u16` | Constant pool `TYPE_DESC` index for the field type. |

Flag bits:

| Bit | Name | Description |
|-----|------|-------------|
| 0 | `PUBLIC` | |
| 1 | `PROTECTED` | |
| 2 | `PRIVATE` | |
| 3 | `STATIC` | |
| 4 | `READONLY` | Property is readonly. |

### Method descriptor

Each method is encoded as:

| Field | Type | Description |
|-------|------|-------------|
| `flags` | `u16` | Bit flags (see below). |
| `name_index` | `u16` | Constant pool `STRING` index for the method name. Constructors use `"<construct>"`, destructors use `"<destruct>"`. |
| `descriptor_index` | `u16` | Constant pool `TYPE_DESC` index for the method signature (parameter and return types). |
| `throws_count` | `u16` | Number of declared checked exceptions. |
| `throws_types` | `u16[]` | Constant pool `CLASS` indices for each declared exception type. |
| `max_locals` | `u16` | Maximum number of local variable slots used (includes `this` for instance methods and all parameters). |
| `max_stack` | `u16` | Maximum operand stack depth. |
| `code_length` | `u32` | Length of the bytecode array in bytes. |
| `code` | `u8[]` | Bytecode instructions. |
| `exception_table_count` | `u16` | Number of exception table entries. |
| `exception_table` | varies | Exception table entries (see [Exception handling](#exception-handling)). |
| `line_table_count` | `u16` | Number of line-number table entries (`0` when no debug info is emitted). |
| `line_table` | varies | Line-number table entries (see below). |

Flag bits:

| Bit | Name | Description |
|-----|------|-------------|
| 0 | `PUBLIC` | |
| 1 | `PROTECTED` | |
| 2 | `PRIVATE` | |
| 3 | `STATIC` | |
| 4 | `CONST` | Method does not modify `this`. |
| 5 | `NODISCARD` | Caller must use return value. |
| 6 | `CONSTRUCTOR` | This is a `construct` method. |
| 7 | `DESTRUCTOR` | This is a `destruct` method. |
| 8 | `ABSTRACT` | Abstract method (no body). |
| 9 | `FINAL` | Final method; cannot be overridden. At link time, the VM **must** reject a subclass that overrides a vtable slot occupied by a method with this flag. |

**Abstract method representation.** A method with the `ABSTRACT` flag has **no code**: `max_locals = 0`,
`max_stack = 0`, `code_length = 0`, an empty `code` array, `exception_table_count = 0`, and
`line_table_count = 0`. The loader **must** reject a module containing an `ABSTRACT` method with
`code_length ≠ 0`, and conversely a non-abstract, non-interface method with `code_length = 0`. Interface method
declarations are encoded the same way (implicitly abstract). `ABSTRACT` and `FINAL` are mutually exclusive.

**Line-number table.** Optional debug information mapping bytecode offsets to source lines, used for
[stack trace construction](#stack-trace-construction). Each entry is:

| Field | Type | Description |
|-------|------|-------------|
| `start_pc` | `u16` | First bytecode offset covered by this entry. |
| `line` | `u32` | 1-based source line number. |

Entries are sorted by ascending `start_pc`; an entry covers the offsets from its `start_pc` up to (excluding) the
next entry's `start_pc` (or the end of the code array for the last entry). Compilers **should** emit the table by
default; a stripped build may omit it (`line_table_count = 0`), in which case stack trace lines are reported as `0`.

---

## Runtime structures

### Heap

The heap stores all dynamically allocated data: objects, arrays, strings, and closures. Each heap allocation
carries a header followed by its payload (see [Object model](#object-model)). The heap is managed by a garbage
collector (see [Garbage collection contract](#garbage-collection-contract)).

### Call frame and operand stack

Each method invocation creates a **call frame** on the VM's call stack. A call frame contains:

| Component | Description |
|-----------|-------------|
| `method` | Reference to the method descriptor being executed. |
| `pc` | Program counter — byte offset into the method's `code` array. |
| `locals` | Array of tagged values, size = `max_locals`. Slot 0 is `this` for instance methods; parameters follow starting at slot 0 (static) or slot 1 (instance). |
| `stack` | Operand stack — array of tagged values, max depth = `max_stack`. |
| `sp` | Stack pointer — index of the next free slot on the operand stack. |

When a method returns, its frame is popped and the return value (if any) is pushed onto the caller's operand
stack.

**Stack overflow.** If creating a new call frame would exceed the VM's maximum call stack depth, the VM throws
`StackOverflowException` (a `RuntimeException`). The maximum depth is implementation-defined but must be
documented by the runtime.

An implementation applying [tail call optimization](optimizations.md#compiler-optimizations) reuses the current
frame for calls in tail position instead of creating a new one, so recursion that overflows without the
optimization may run indefinitely with it. This divergence is permitted — depth is implementation-defined and
`StackOverflowException` is not a control-flow mechanism — but a runtime that applies the optimization should say
so where it documents its maximum depth. See
[optimizations.md § Tail call optimization and `StackOverflowException`](optimizations.md#tail-call-optimization-and-stackoverflowexception).

### Static storage

Each class has a **static storage area** holding one slot per `static` field. Static fields are initialized to
their [default values](specs.md#null-initialization-and-default-values) when the class is first loaded. If the
class has a static initializer (field declarations with explicit values), the VM executes it before the first use
of the class.

---

## Object model

### Object layout

Every heap-allocated object consists of:

```
┌─────────────────────┐
│  Object Header      │
│  ├─ class_id : u32  │  Index into the VM's class table.
│  ├─ flags    : u32  │  Bit 0 = readonly class.
│  └─ gc_info  : …    │  Implementation-defined GC metadata.
├─────────────────────┤
│  Fields             │
│  ├─ field_0 : Value │  Ordered by declaration order.
│  ├─ field_1 : Value │
│  └─ …               │
└─────────────────────┘
```

Fields are laid out in the order they appear in the class definition. Inherited fields come first (in parent
declaration order), then the fields declared in the current class. The total field count is the sum of all
inherited and local fields.

### Array layout

```
┌──────────────────────────┐
│  Array Header            │
│  ├─ element_type : u8    │  Tag of element type (1–5).
│  ├─ length       : u32   │  Number of elements (immutable after creation).
│  └─ gc_info      : …     │
├──────────────────────────┤
│  Elements                │
│  ├─ element_0 : Value    │
│  ├─ element_1 : Value    │
│  └─ …                    │
└──────────────────────────┘
```

Arrays are **fixed-size** after creation. Index access outside `[0, length-1]` throws
`IndexOutOfBoundsException`.

**Multidimensional arrays** (`T[][]`, `T[][][]`, etc.) are represented as nested arrays. The outer array's `element_type` tag is `5` (reference) and each element points to an inner array object on the heap. The `TYPE_DESC` constant pool entry for the outer array's element type is the inner type descriptor (e.g. `"int[]"` for a `int[][]`, `"int[][]"` for a `int[][][]`).

### String representation

Strings are immutable heap objects. Internally a string stores:

| Field | Description |
|-------|-------------|
| `length` | Number of characters (not bytes). |
| `data` | UTF-8 encoded byte sequence. |

String interning: the VM **may** intern string literals from the constant pool so that identical literals share
the same object. Interning is an optimization; correctness must not depend on it (string equality is always
by content, never by reference identity). See [optimizations.md § VM optimizations](optimizations.md#vm-optimizations).

### Enum representation

Enums are compiled to classes:

- **Basic enums** (no backing type): each case is a `static readonly` integer field (0, 1, 2, …) on the
  generated class. Comparisons use integer equality.
- **Typed enums** (`enum E: int` or `enum E: string`): each case is a `static readonly` field of the backing
  type. The `.value` property reads this field. `from()` / `tryFrom()` are compiled as static methods with a
  lookup table or chain of comparisons.
- **Custom methods and properties** (see [specs.md § Custom methods and properties](specs.md#enum-custom-methods-and-properties)): compiled like class methods and fields on the generated enum class.

---

## Instruction set

### Notation

Each instruction is described with:

- **Name** — mnemonic.
- **Operands** — bytes following the opcode, with types: `u8` (unsigned byte), `i8` (signed byte), `u16`
  (unsigned 16-bit, big-endian), `i16` (signed 16-bit, big-endian).
- **Stack** — effect on the operand stack, written `[before → after]`. `…` denotes values already on the
  stack that are not affected.
- **Description** — semantics.

### Stack manipulation

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `NOP` | — | `[… → …]` | No operation. |
| `POP` | — | `[…, value → …]` | Discard top value. |
| `DUP` | — | `[…, value → …, value, value]` | Duplicate top value. |
| `SWAP` | — | `[…, a, b → …, b, a]` | Swap top two values. |

### Constants

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `CONST_NULL` | — | `[… → …, null]` | Push null. |
| `CONST_TRUE` | — | `[… → …, true]` | Push boolean true. |
| `CONST_FALSE` | — | `[… → …, false]` | Push boolean false. |
| `CONST_IZERO` | — | `[… → …, 0]` | Push int 0. |
| `CONST_IONE` | — | `[… → …, 1]` | Push int 1. |
| `CONST_FZERO` | — | `[… → …, 0.0]` | Push float 0.0. |
| `CONST_FONE` | — | `[… → …, 1.0]` | Push float 1.0. |
| `BIPUSH` | `i8` | `[… → …, int]` | Push `i8` value as int (sign-extended to 64 bits). |
| `SIPUSH` | `i16` | `[… → …, int]` | Push `i16` value as int (sign-extended to 64 bits). |
| `LDC` | `u16` | `[… → …, value]` | Load constant from pool at index `u16`. Pushes int, float, or string reference depending on the entry type. |

### Local variable access

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `LOAD` | `u16` | `[… → …, value]` | Push the value of local variable at index `u16`. |
| `STORE` | `u16` | `[…, value → …]` | Pop value and store it into local variable at index `u16`. |
| `LOAD_0` | — | `[… → …, value]` | Push local 0 (shortcut, typically `this`). |
| `LOAD_1` | — | `[… → …, value]` | Push local 1. |
| `LOAD_2` | — | `[… → …, value]` | Push local 2. |
| `LOAD_3` | — | `[… → …, value]` | Push local 3. |
| `STORE_0` | — | `[…, value → …]` | Store into local 0. |
| `STORE_1` | — | `[…, value → …]` | Store into local 1. |
| `STORE_2` | — | `[…, value → …]` | Store into local 2. |
| `STORE_3` | — | `[…, value → …]` | Store into local 3. |

### Integer arithmetic

All integer operations work on `int` (64-bit signed) values. `byte` values are widened to `int` before
arithmetic (see `B2I`).

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `IADD` | — | `[…, a, b → …, a+b]` | Integer addition. Overflow wraps (two's complement). |
| `ISUB` | — | `[…, a, b → …, a-b]` | Integer subtraction. Overflow wraps (two's complement). |
| `IMUL` | — | `[…, a, b → …, a*b]` | Integer multiplication. Overflow wraps (two's complement). |
| `IDIV` | — | `[…, a, b → …, a/b]` | Integer division (truncation toward zero). Throws `ArithmeticException` if `b` is zero. |
| `IMOD` | — | `[…, a, b → …, a%b]` | Integer remainder. Throws `ArithmeticException` if `b` is zero. |
| `INEG` | — | `[…, a → …, -a]` | Integer negation. |
| `IINC` | `u16` index, `i16` delta | `[… → …]` | Increment local variable at `index` by `delta`. Does not touch the operand stack. |

### Float arithmetic

All float operations work on `float` (64-bit IEEE 754) values.

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `FADD` | — | `[…, a, b → …, a+b]` | Float addition. |
| `FSUB` | — | `[…, a, b → …, a-b]` | Float subtraction. |
| `FMUL` | — | `[…, a, b → …, a*b]` | Float multiplication. |
| `FDIV` | — | `[…, a, b → …, a/b]` | Float division. Division by zero produces IEEE 754 infinity or NaN. |
| `FMOD` | — | `[…, a, b → …, a%b]` | Float remainder (IEEE 754). |
| `FNEG` | — | `[…, a → …, -a]` | Float negation. |

### Type conversions

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `I2F` | — | `[…, int → …, float]` | Convert int to float. |
| `F2I` | — | `[…, float → …, int]` | Convert float to int (truncation toward zero). Values outside `int` range are clamped to `INT_MIN` / `INT_MAX`. |
| `I2B` | — | `[…, int → …, byte]` | Convert int to byte (keep low 8 bits). |
| `B2I` | — | `[…, byte → …, int]` | Convert byte to int (zero-extend). |
| `TO_STRING` | — | `[…, value → …, string]` | Convert top of stack to its string representation. For `int`: decimal digits. For `float`: decimal notation. For `bool`: `"true"` or `"false"`. For `byte`: decimal digits (0–255). For `null`: throws `NullPointerException`. For references: calls `toString()` if the object implements `Stringable`; otherwise a compile-time error should have prevented reaching this opcode. |

### Comparison

All comparison instructions pop their operands and push a `bool` result, except `CMP_THREE_WAY` which pushes
an `int`.

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `CMP_EQ` | — | `[…, a, b → …, bool]` | Equality. For `int`, `float`, `byte`, `bool`: value equality. For `string`: content equality. For references: **reference identity** (same object). For value-based equality of objects, use the [ValueEquatable](specs.md#valueequatable-interface) interface (`valueEquals` / `valueHash`). For `null`: `null == null` is `true`; `null == non-null` is `false`. |
| `CMP_NE` | — | `[…, a, b → …, bool]` | Not-equal (inverse of `CMP_EQ`). |
| `CMP_LT` | — | `[…, a, b → …, bool]` | Less-than. Defined for `int`, `float`, `byte` (numeric comparison) and `string` (lexicographic). |
| `CMP_GT` | — | `[…, a, b → …, bool]` | Greater-than. |
| `CMP_LE` | — | `[…, a, b → …, bool]` | Less-than-or-equal. |
| `CMP_GE` | — | `[…, a, b → …, bool]` | Greater-than-or-equal. |
| `CMP_THREE_WAY` | — | `[…, a, b → …, int]` | Spaceship operator (`<=>`). Pushes `−1` if `a < b`, `0` if `a == b`, `1` if `a > b`. Same type rules as `CMP_LT`. |
| `IS_NULL` | — | `[…, value → …, bool]` | Push `true` if value is `null`. |
| `IS_NONNULL` | — | `[…, value → …, bool]` | Push `true` if value is not `null`. |
| `NOT` | — | `[…, bool → …, bool]` | Logical negation. |

### Control flow

Branch offsets are relative to the address of the **opcode** of the branch instruction.

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `IF_TRUE` | `i16` offset | `[…, bool → …]` | Branch if top is `true`. |
| `IF_FALSE` | `i16` offset | `[…, bool → …]` | Branch if top is `false`. |
| `GOTO` | `i16` offset | `[… → …]` | Unconditional branch. |
| `GOTO_W` | `i32` offset | `[… → …]` | Wide unconditional branch (for methods with code > 32 KiB). |

> **Wide conditional branches.** There are deliberately no `IF_TRUE_W` / `IF_FALSE_W` opcodes. When a conditional
> branch target is out of `i16` range (beyond ±32 KiB), the compiler **must** emit the inverted condition jumping
> over a wide unconditional branch:
>
> ```
> IF_FALSE   NEXT        // inverted condition, short offset
> GOTO_W     FAR_TARGET  // wide jump to the real target
> NEXT:
> ```

### Object operations

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `NEW` | `u16` class_index | `[… → …, ref]` | Allocate a new object of the class referenced by constant pool entry `class_index`. All fields are set to their [default values](specs.md#null-initialization-and-default-values). Pushes the reference. Does **not** call a constructor (the compiler emits a separate `INVOKE_SPECIAL` for that). |
| `INSTANCEOF` | `u16` class_index | `[…, ref → …, bool]` | Push `true` if the object is an instance of the given class (or a subclass), `false` otherwise. `null` always produces `false`. |
| `CHECKCAST` | `u16` class_index | `[…, ref → …, ref]` | If the object is an instance of the given class (or a subclass), leave the reference on the stack. Otherwise throw `InvalidCastException`. `null` passes the check (null can be cast to any reference type). |

### Array operations

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `NEW_ARRAY` | `u16` type_index | `[…, size → …, ref]` | Allocate an array of `size` elements. `type_index` is a constant pool `TYPE_DESC` entry for the element type. Elements are initialized to their default values. |
| `NEW_ARRAY_INIT` | `u16` type_index, `u16` count | `[…, e₀, e₁, …, eₙ₋₁ → …, ref]` | Create an array from `count` values on the stack (e₀ is the first element). Used for `new T[]{ ... }` initializer lists. |
| `ARRAY_LOAD` | — | `[…, ref, index → …, value]` | Load element at `index` from array `ref`. Throws `IndexOutOfBoundsException` if out of range. |
| `ARRAY_STORE` | — | `[…, ref, index, value → …]` | Store `value` at `index` in array `ref`. Throws `IndexOutOfBoundsException` if out of range. |
| `ARRAY_LENGTH` | — | `[…, ref → …, int]` | Push the length of the array. |

> **Multidimensional arrays.** `new T[n₁][n₂]…[nₖ]` is compiled as a series of `NEW_ARRAY` instructions combined with loops and `ARRAY_STORE` — no dedicated multidimensional opcode exists. See [compiler.md § Multidimensional array creation](compiler.md#multidimensional-array-creation) for the full desugaring.

### Field access

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `GET_FIELD` | `u16` field_ref | `[…, ref → …, value]` | Read instance field. `field_ref` is a constant pool `FIELD_REF` index. Throws `NullPointerException` if `ref` is `null`. |
| `SET_FIELD` | `u16` field_ref | `[…, ref, value → …]` | Write instance field. Throws `NullPointerException` if `ref` is `null`. The VM **must** reject writes to `readonly` fields outside constructors at runtime (as a safety net; the compiler should have caught this). |
| `GET_STATIC` | `u16` field_ref | `[… → …, value]` | Read static field. |
| `SET_STATIC` | `u16` field_ref | `[…, value → …]` | Write static field. |

### Method invocation

All invoke instructions pop arguments from the stack (rightmost argument on top), execute the method, and push
the return value (if non-void). For instance methods, the receiver (`this`) is popped *before* the arguments
(i.e. it is below the arguments on the stack).

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `INVOKE_STATIC` | `u16` method_ref | `[…, arg₀, arg₁, … → …, result?]` | Call a static method. |
| `INVOKE_INSTANCE` | `u16` method_ref | `[…, receiver, arg₀, arg₁, … → …, result?]` | Call an instance method with **virtual dispatch** (vtable lookup on the runtime class of `receiver`). Throws `NullPointerException` if `receiver` is `null`. |
| `INVOKE_SPECIAL` | `u16` method_ref | `[…, receiver, arg₀, arg₁, … → …, result?]` | Call an instance method **without** virtual dispatch. Used for constructors (`<construct>`), `super` calls, and `private` methods. |
| `INVOKE_CLOSURE` | `u8` arg_count | `[…, closure, arg₀, arg₁, … → …, result?]` | Call a closure object. `arg_count` is the number of arguments. The VM invokes the closure's internal method with the captured environment and the provided arguments. |
| `RETURN` | — | `[… → ]` | Return from a `void` method. The current frame is discarded. |
| `RETURN_VALUE` | — | `[…, value → ]` | Pop value, discard the current frame, and push `value` onto the caller's operand stack. |

### String operations

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `STR_CONCAT` | — | `[…, str₁, str₂ → …, result]` | Concatenate two strings. Both operands must be string references. The result is a new string. |

The compiler uses `TO_STRING` to convert non-string operands before `STR_CONCAT` (see
[String concatenation](#string-concatenation)).

### Exception operations

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `THROW` | — | `[…, exception_ref → ]` | Throw the exception. The VM begins [stack unwinding](#throw-and-stack-unwinding). Throws `NullPointerException` if `exception_ref` is `null`. |

### Miscellaneous

| Name | Operands | Stack | Description |
|------|----------|-------|-------------|
| `DUP_X1` | — | `[…, a, b → …, b, a, b]` | Duplicate top value and insert below second value. Useful for compound operations. |

---

## Method dispatch

### Static methods

`INVOKE_STATIC` resolves the method at load time (or on first call). The `method_ref` constant pool entry
identifies the class and method name + descriptor. The VM looks up the method in the target class and calls
it directly. No receiver is needed.

### Instance methods

`INVOKE_INSTANCE` uses **virtual dispatch**:

1. Pop the receiver reference from the stack.
2. Determine the **runtime class** of the receiver.
3. Look up the method in the runtime class's **vtable** using the method's vtable index.
4. Create a new frame; store the receiver in local 0 (`this`), arguments in locals 1, 2, ….

The vtable is an array of method pointers, one per non-static, non-private method. When a class extends
another, it inherits the parent's vtable entries and overrides slots for methods it redefines. The vtable
index for each method is determined at class load time.

All non-static, non-private instance methods participate in virtual dispatch by default (see [specs.md § Virtual method dispatch](specs.md#virtual-method-dispatch)). Abstract methods have no implementation in the declaring class; concrete subclasses provide the implementation. Final methods cannot be overridden; the compiler may optimize calls to final methods by bypassing the vtable when the receiver's static type is known (see [optimizations.md § Compiler optimizations](optimizations.md#compiler-optimizations)).

### Constructors

Constructors are invoked with `INVOKE_SPECIAL`. The typical sequence for `new MyClass(args)` is:

```
NEW           <MyClass>           // allocate, push ref
DUP                               // duplicate ref (one for constructor, one as result)
LOAD_1                            // push first arg
LOAD_2                            // push second arg
INVOKE_SPECIAL <MyClass.<construct>(int, string) -> void>
// ref remains on stack as the result of the `new` expression
```

Constructors that call `super(...)` emit `INVOKE_SPECIAL` targeting the parent's constructor as the first
statement in the constructor body. Likewise, a constructor that delegates with `this(...)` (see
[specs.md § Constructor chaining](specs.md#constructor-chaining-this)) emits `INVOKE_SPECIAL` targeting the
sibling constructor of the **same** class as its first instruction sequence (`LOAD_0` for the receiver, the
arguments, then `INVOKE_SPECIAL <MyClass.<construct>(…)>`). The compiler guarantees delegation chains are acyclic
(E046); the VM does not re-check at runtime.

### Super calls

`super.method(args)` compiles to `INVOKE_SPECIAL` targeting the parent class's method. This bypasses the
vtable and calls the parent's implementation directly.

### Interface dispatch

Interfaces are dispatched through `INVOKE_INSTANCE`. At class load time, the VM resolves interface method
references to vtable indices in the concrete class. If a class implements interface `I` with method `m()`,
the vtable slot for `I.m()` maps to the class's implementation.

---

## Exception handling

### Exception table

Each method contains an exception table with zero or more entries. Each entry has:

| Field | Type | Description |
|-------|------|-------------|
| `start_pc` | `u16` | Start of the protected range (inclusive). |
| `end_pc` | `u16` | End of the protected range (exclusive). |
| `handler_pc` | `u16` | Address of the handler code. |
| `catch_type` | `u16` | Constant pool `CLASS` index of the caught exception type. `0` means catch-all (`finally`). |

Entries are ordered by specificity: more specific catch types first, then more general, then finally.

### Throw and stack unwinding

When `THROW` is executed (or an implicit exception occurs, e.g. `IDIV` by zero):

1. The VM records the current exception object.
2. Starting from the current frame, the VM searches the exception table for an entry where:
   - `start_pc ≤ current pc < end_pc`, and
   - the exception's class is the same as or a subclass of `catch_type` (or `catch_type` is `0`).
3. If a matching entry is found: clear the operand stack, push the exception reference, and set `pc` to
   `handler_pc`.
4. If no matching entry is found in the current frame: pop the frame and repeat from step 2 in the caller's
   frame.
5. If no frame handles the exception: the program terminates with an unhandled exception error. The VM prints
   the exception message and stack trace to stderr, then exits with a non-zero exit code.

### Finally blocks

`finally` blocks are compiled using exception table entries with `catch_type = 0`. The compiler duplicates
the finally code at every exit point of the try block (normal completion, explicit `return`, `break`,
`continue`) and also installs a catch-all handler that executes the finally block and re-throws the exception.

### Stack trace construction

The stack trace is captured **during the execution of the base `Exception` constructor**: when
`Exception.<construct>(string)` runs (invoked directly or through the `super(...)` chain of a subclass
constructor), the VM natively walks the current call stack and assigns the resulting `ExecutionPoint[]` to the
`stackTrace` field **before the constructor returns**. Frames belonging to the exception hierarchy's own
constructor chain are excluded, so the trace starts at the `new` site.

Because the assignment happens inside `construct`, it follows the ordinary `readonly` rule for the
`class readonly Exception` (see [specs.md § Exception class hierarchy](specs.md#exception-class-hierarchy)) —
the VM needs **no mechanism to bypass readonly enforcement**.

Each `ExecutionPoint` contains:

- `line`: the source line number, derived from the [line-number table](#method-descriptor) of the method
  (`line_table`), by selecting the entry covering the frame's current `pc`. If the table is absent
  (`line_table_count = 0`), `line` is `0`.
- `file`: the source file path (derived from the module's class name and namespace).

**Optimized builds.** The walk only sees frames that exist. Inlining and tail call optimization remove frames an
unoptimized build would have created, so the number and identity of the entries in `stackTrace` are
implementation-defined: a VM may report elided frames when the compiler supplies inline metadata, but is not
required to. Three properties must hold at every optimization level — the trace is never empty, frames are
ordered innermost first, and no frame is reported for a call the program does not make. See
[optimizations.md § Stack traces and frame-eliding optimizations](optimizations.md#stack-traces-and-frame-eliding-optimizations).

---

## Closures and anonymous functions

Anonymous functions are compiled to **closure objects**. The compiler generates a synthetic class for each
closure, containing:

1. **Captured fields** — one field per captured variable from the enclosing scope.
2. **An invoke method** — the body of the anonymous function, with captured variables accessed through `this`.

### Variable capture and boxing

Variables captured by a closure are **boxed** if they can be mutated after capture (by the enclosing scope or
by the closure itself). A box is a single-element wrapper object (`Box<T>`) with a mutable `value` field.
Both the enclosing scope and the closure reference the same box.

Variables captured **read-only** (e.g. `const` variables, or variables never reassigned after capture) may be
copied directly into the closure's fields without boxing.

### Invocation

`INVOKE_CLOSURE` pops the closure reference and the arguments from the stack, then calls the closure's invoke
method. The closure's captured fields are accessible via `this` inside the invoke method.

The compiler determines the closure's type signature at compile time. A closure assigned to a `typedef`'d
function type is type-compatible if the parameter and return types match.

---

## Compilation strategies

This section describes how the compiler translates high-level language features into bytecode. These are
**normative** — an implementation must produce equivalent results, but may use different instruction sequences
as long as the observable behavior is the same.

### Templates (monomorphization)

Templates are instantiated at **compile time**. When a template class or method is used with concrete type
arguments, the compiler generates a **specialized version** with those types substituted:

- `Vector<int>` and `Vector<float>` produce two separate classes in the module output, each with its own
  fields, methods, and vtable.
- Template method `Utils.max<int>` and `Utils.max<float>` produce two separate static methods.

Bounded type parameters (`template <type T extends Bound>`) are enforced at compile time only. The compiler
rejects instantiations where the concrete type does not satisfy the bound (see [specs.md § Bounded type parameters](specs.md#bounded-type-parameters), [compiler.md § Template instantiation](compiler.md#template-instantiation)). No bound metadata survives into bytecode.

No template metadata survives into bytecode. The VM does not know about user-defined templates.

**Native template classes** (`system.List<T>`, `system.Map<K,V>`, `system.MapEntry<K,V>`) are a special
case: these are provided by the runtime, not compiled from NL source. The compiler emits monomorphized class
names in constant pool references (e.g. `"system.List<int>"`, `"system.Map<string, int>"`,
`"system.MapEntry<string, int>"`). The VM recognizes these as native template instantiations and provides
type-appropriate implementations. Internally, the native implementation may use a single generic
implementation with tagged values (since all VM values are already tagged), but this is an implementation
detail.

For **`system.Map<K,V>`**, key lookup uses:
- **Primitives** (`int`, `float`, `bool`, `byte`) and **`string`**: built-in value equality.
- **Reference types implementing ValueEquatable**: `valueEquals(other)` for equality, `valueHash()` for hashing (see [specs.md § ValueEquatable interface](specs.md#valueequatable-interface)).
- **Other reference types**: reference identity (same object instance).

### Ref parameters (boxing)

When a parameter is declared `ref`, the compiler wraps the caller's variable in a **box** (a single-field
object). The callee reads and writes through the box. After the call, the caller unboxes the updated value.

Compilation of `Utils.swap(ref x, ref y)`:

```
// Caller side:
NEW           <Box<int>>         // box for x
DUP
LOAD          x_local
SET_FIELD     <Box.value>
STORE         box_x              // local holding the box

NEW           <Box<int>>         // box for y
DUP
LOAD          y_local
SET_FIELD     <Box.value>
STORE         box_y

LOAD          box_x
LOAD          box_y
INVOKE_STATIC <Utils.swap(Box<int>, Box<int>) -> void>

LOAD          box_x              // unbox x
GET_FIELD     <Box.value>
STORE         x_local

LOAD          box_y              // unbox y
GET_FIELD     <Box.value>
STORE         y_local
```

`const ref` parameters use the same boxing but the compiler ensures no writes to `Box.value` inside the callee.

### String concatenation

The expression `"value: " + n` (where `n` is `int`) compiles to:

```
LDC           "value: "
LOAD          n_local
TO_STRING                         // int → string
STR_CONCAT                        // "value: " + "42"
```

For multi-part concatenation like `a + b + c`, the compiler chains `TO_STRING` and `STR_CONCAT` pairs
left-to-right.

### Union types at runtime

Union types (`T|null`, `Type1|Type2|null`) require no special VM support. Since all values are tagged, a
variable of type `string|null` simply holds either a string reference (tag `5`) or null (tag `0`). The compiler
emits the appropriate type checks (using `IS_NULL`, `INSTANCEOF`, or tag comparisons) when narrowing a union
type.

### For-each loops

`for (auto item : collection)` and `for (const auto item : collection)` are desugared identically by the compiler. The `const` qualifier affects only compile-time checks (see [specs.md § Loops](specs.md#loops), [compiler.md § For-each loop in const context](compiler.md#for-each-loop-in-const-context)); the bytecode pattern is the same. The `ARRAY_LOAD` + `STORE` sequence yields copy semantics: for value types, a copy of the value; for reference types, a copy of the reference (see [specs.md § Loops](specs.md#loops)).

- **For arrays (`T[]`)**: the compiler generates an index-based loop:
  ```
  // for (const auto item : arr)
  CONST_IZERO
  STORE        i_local
  LOOP_START:
  LOAD         i_local
  LOAD         arr_local
  ARRAY_LENGTH
  CMP_LT
  IF_FALSE     LOOP_END
  LOAD         arr_local
  LOAD         i_local
  ARRAY_LOAD
  STORE        item_local
  // ... loop body ...
  IINC         i_local, 1
  GOTO         LOOP_START
  LOOP_END:
  ```
- **For `system.List<T>`**: same pattern using `list.size()` and `list.get(i)` via `INVOKE_INSTANCE`.
- **For `system.Map<K,V>`**: the compiler calls `entries()` to obtain a `MapEntry<K,V>[]`, then iterates
  over the resulting array with an index-based loop:
  ```
  // for (const auto entry : map)
  LOAD         map_local
  INVOKE_INSTANCE <system.Map.entries() -> MapEntry[]>
  STORE        entries_local
  CONST_IZERO
  STORE        i_local
  LOOP_START:
  LOAD         i_local
  LOAD         entries_local
  ARRAY_LENGTH
  CMP_LT
  IF_FALSE     LOOP_END
  LOAD         entries_local
  LOAD         i_local
  ARRAY_LOAD
  STORE        entry_local
  // ... loop body (entry.key via GET_FIELD, entry.value via GET_FIELD) ...
  IINC         i_local, 1
  GOTO         LOOP_START
  LOOP_END:
  ```

### Named and optional parameters

Named and optional parameters are resolved entirely at compile time:

- The compiler reorders arguments to match the parameter order.
- Missing optional parameters are filled with their default values (emitted as constant-loading instructions).
- At the bytecode level, every call site passes all parameters in declaration order — no special VM support.

### Increment and decrement operators

`++` and `--` (prefix and postfix) are compiled differently depending on context:

- **On a local int variable:** `IINC` is used for the side effect. For **prefix** (`++x` as an expression),
  `IINC` followed by `LOAD`; for **postfix** (`x++` as an expression), `LOAD` followed by `IINC` (the
  pre-increment value remains on the stack).
- **On float or byte locals:** the compiler emits `LOAD`, then `CONST_FONE`/`CONST_IONE` + `FADD`/`IADD`
  (with `I2B` for byte), then `STORE`. Prefix vs postfix determines whether the load happens before or
  after the computation.
- **On object fields or array elements:** the compiler emits a `GET_FIELD`/`ARRAY_LOAD`, performs the
  increment, and emits `SET_FIELD`/`ARRAY_STORE`. For postfix, a `DUP` before the increment preserves the
  original value.
- **Overloaded `++`/`--`:** if the type defines `operator++` or `operator--`, the compiler emits
  `INVOKE_INSTANCE` instead.

### Operator overloading

Overloaded operators compile to `INVOKE_INSTANCE` or `INVOKE_STATIC` calls. For example, `p1 + p2` where `+`
is overloaded on `Vector2`:

```
LOAD          p1_local            // receiver
LOAD          p2_local            // argument
INVOKE_INSTANCE <Vector2.operator+(Vector2) -> Vector2>
```

Compound assignment operators like `p3 += v` similarly compile to `INVOKE_INSTANCE` on `operator+=`.

Built-in operators on primitives (`int + int`, `float * float`) use the arithmetic opcodes (`IADD`, `FMUL`,
etc.) and are **not** method calls.

### Nullish coalescing and elvis operators

`a ?? b` (nullish coalescing) compiles to:

```
LOAD          a_local
DUP
IS_NONNULL
IF_TRUE       SKIP_DEFAULT
POP
// ... evaluate b ...
SKIP_DEFAULT:
```

`a ?: b` (elvis) compiles similarly, but the falsy check tests for `null`, `false`, and `0`:

```
LOAD          a_local
DUP
// check falsy (null, false, or 0)
DUP
IS_NULL
IF_TRUE       IS_FALSY
DUP
CONST_FALSE
CMP_EQ
IF_TRUE       IS_FALSY
DUP
CONST_IZERO
CMP_EQ
IF_TRUE       IS_FALSY
GOTO          NOT_FALSY
IS_FALSY:
POP
// ... evaluate b ...
GOTO          DONE
NOT_FALSY:
DONE:
```

The compiler may simplify based on the static type of `a` (e.g. if `a` is `int`, only check for `0`).

### Switch statements

`switch/case` with fall-through semantics compiles to a chain of comparisons. Each `case` label is a
comparison + conditional branch to the case body. `break` compiles to `GOTO` targeting the instruction
after the switch block. Without `break`, execution falls through to the next case body.

```
LOAD          value_local
DUP
CONST_IONE                        // case 1
CMP_EQ
IF_FALSE      CASE_2
POP
// ... doSomething() ...
GOTO          END                 // break
CASE_2:
DUP
BIPUSH        2                   // case 2
CMP_EQ
IF_FALSE      DEFAULT
POP
// ... doSomethingElse() ...
GOTO          END                 // break
DEFAULT:
POP
// ... doNothing() ...
// break (or fall through to END)
END:
```

### Match expressions

`match(value) { ... }` compiles to a chain of comparisons and branches:

```
LOAD          value_local
DUP
SIPUSH        200
CMP_EQ
IF_FALSE      CASE_2
POP
LDC           "ok"
GOTO          END
CASE_2:
DUP
SIPUSH        404
CMP_EQ
IF_FALSE      DEFAULT
POP
LDC           "not found"
GOTO          END
DEFAULT:
POP
INVOKE_STATIC <Config.getDefaultValue() -> string>
END:
```

The compiler guarantees exhaustiveness at compile time (see [compiler.md § Match exhaustiveness](compiler.md#match-exhaustiveness)):
either a `default` arm exists, or the arms cover the whole value domain (enum cases, `bool`). The generated code
therefore always reaches exactly one arm — no runtime "unmatched value" path is emitted.

---

## Program startup

1. **Load modules.** The VM loads the module containing the `main` method and all transitively referenced
   modules.
2. **Link.** Resolve all constant pool references (class, method, field) across modules. Build vtables,
   static storage, and string intern pool.
3. **Initialize statics.** Execute static field initializers for each class, in dependency order (a class's
   initializer runs before any code that references it).
4. **Create main thread.** Allocate a thread (see [Threading model](#threading-model)).
5. **Build arguments.** Construct a `string[]` array from command-line arguments.
6. **Invoke main.** Create a frame for `main(string[])` with `args` in local 0.
   Begin execution.
7. **Exit.** When `main` returns, use the returned `int` value as the process exit code. If an unhandled
   exception propagates out of `main`, print the exception and stack trace to stderr and exit with code `1`.

---

## VM invocation (nlvm)

The VM is invoked as:

    nlvm [options] <module-or-program> [--] [program args...]

### Arguments

| Argument | Description |
|----------|-------------|
| `<module-or-program>` | Path to the main module (`.nlm`) or to a program bundle (implementation-defined). This module (or the designated entry in the bundle) must contain the `main(string[])` entry point (see [specs.md § Entry point](specs.md#entry-point)). |
| `[program args...]` | Optional arguments passed to the NL program as `args` (after `args[0]`, the program name). The VM builds `args` as described in [Program startup](#program-startup). |

If `--` is used, everything after it is passed as program arguments; otherwise, the first unrecognized non-option argument starts the module path and subsequent arguments are program args.

### Options

| Option | Description |
|--------|-------------|
| `--version` | Print VM version and exit. |
| `-h`, `--help` | Print usage and exit. |
| `-v`, `--verbose` | Verbose execution (e.g. module loading, implementation-defined). An implementation **should** report each loaded module's `version` and `opt_level` (`unrecorded` for a version ≤ 2 module), so that a mixed-level build is identifiable — see [Module format](#module-format). |
| `-D<path>`, `--module-path <path>` | *(Optional)* Additional directory or list of directories for resolving module dependencies (implementation-defined). |

### Exit codes

- The VM exits with the return value of `main` (see [Program startup](#program-startup)).
- If an unhandled exception propagates out of `main`, the VM prints the exception and stack trace to stderr and exits with code `1`.

---

## Standard library binding

Standard library classes (`system.Out`, `system.io.File`, `system.thread.Thread`, etc.) are implemented as
**native modules**. Their methods are not compiled to NL bytecode; instead, the VM provides built-in
implementations that interface with the host operating system.

At load time, the VM recognizes references to `system.*` classes and binds them to native implementations.
From the bytecode's perspective, calling `system.Out.print(s)` is an `INVOKE_STATIC` like any other — the
VM intercepts the call and runs the native code.

Native method binding is determined by the fully qualified class name and method descriptor. An implementation
must provide native bindings for all methods listed in [stdlib.md](stdlib.md).

String instance methods (`length()`, `charAt()`, `substring()`, etc.) are also native: `INVOKE_INSTANCE` on
a string reference dispatches to built-in string operations.

Array built-in methods (`slice()`, `map()`, `filter()`, `forEach()`, `sort()`, `find()`) are native methods
on array objects. `ARRAY_LENGTH` is a dedicated opcode for `length()` (performance-critical). The other six
methods are invoked via `INVOKE_INSTANCE` on the array reference and dispatched to native implementations by
the VM. Methods that accept callbacks (`map`, `filter`, `forEach`, `sort`, `find`) receive a closure object
as an argument; the native implementation calls `INVOKE_CLOSURE` internally for each element.

`system.List<T>` and `system.Map<K,V>` instance methods (`size`, `get`, `set`, `remove`, `contains`, `keys`, `values`, `entries`,
`forEach`, etc.) are dispatched via `INVOKE_INSTANCE` on the native object reference. `Map.forEach` receives
a closure with two parameters (key, value); the native implementation calls `INVOKE_CLOSURE` for each entry.
`Map.entries()` returns a native array of `MapEntry<K,V>` objects (see [stdlib.md § Result types](stdlib.md#result-types)).

---

## Threading model

Each `system.thread.Thread` instance corresponds to a separate **VM thread** with its own call stack. The
operand stack and local variables are per-thread and not shared. Heap objects (including static fields) are
shared across all threads.

Memory visibility rules:

- **Mutex** (`system.thread.Mutex`): acquiring a mutex establishes a happens-before relationship with the
  previous release of the same mutex. All writes made before a `unlock()` are visible to the thread that
  subsequently calls `lock()` on the same mutex.
- **Thread join**: all writes made by a thread are visible to the thread that calls `join()` after the
  joined thread completes.
- **Volatile / atomic access**: not currently specified. Accesses to shared fields without synchronization
  may produce stale or inconsistent values. The VM is not required to provide sequential consistency for
  unsynchronized accesses.

Interruption:

- Each VM thread carries an **interrupt flag**, set by `Thread.interrupt()` from any thread and consulted only
  by the interruption points listed in
  [stdlib.md § system.thread.Thread](stdlib.md#systemthreadthread) (`join()`, `join(int)`, `sleep(int)`).
  Writing the flag must be visible to the target thread without additional synchronization by the program.
- A thread already blocked at an interruption point when its flag is set **must** be woken up and made to throw
  `InterruptedException` promptly, rather than left to wait out the thread or the timeout it was blocked on.
- The VM **must not** terminate a thread by force: interruption is a request the target thread observes, and a
  task that never reaches an interruption point runs to completion.

---

## Garbage collection contract

The VM **must** automatically reclaim heap objects that are no longer reachable from any thread's call stack
or from static fields. The specific algorithm (reference counting, mark-and-sweep, generational, etc.) is
implementation-defined.

**Destructor guarantees:**

- If a class defines a `destruct` method, the VM **must** call it before reclaiming the object's memory.
- A destructor is called **at most once** per object.
- The order in which destructors run for unreachable objects is implementation-defined.
- Destructors must not throw exceptions. If a destructor throws, the exception is silently discarded and
  the object is reclaimed.
- The VM **should** call destructors promptly (e.g. at the end of the scope for stack-allocated references),
  but is not required to do so immediately. An implementation that defers destructor calls to a GC cycle
  is conformant.

---

## Object lifecycle

Object lifetime is managed entirely by the garbage collector. There is no `delete` keyword or manual destruction. The VM reclaims heap objects when they become unreachable. Destructors (`destruct`) are called before reclamation; see [Garbage collection contract](#garbage-collection-contract).

Object cloning is provided via the **Cloneable** interface and the `clone()` method (see [specs.md § Cloneable interface](specs.md#cloneable-interface)). There is no dedicated `clone` keyword.
