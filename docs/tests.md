# NL test format

This document describes the format of acceptance tests for the NL language and toolchain. Tests live in the `tests/` directory as YAML files and are intended for use by a compiler, VM, or test runner. The repository does not include a test runner implementation.

## File naming

Test files follow the pattern `m{N}_{XXXX}_{name}.yaml`:

| Part | Meaning | Example |
|------|---------|---------|
| `m{N}` | Milestone number (1–9) from [milestones.md](milestones.md) | `m4` = VM core |
| `{XXXX}` | 4-digit index with spacing (e.g. 0010, 0020, 0030) | Allows insertion without renumbering |
| `{name}` | Descriptive name (lowercase, underscores) | `minimal_main`, `class_instantiation` |

**Examples:** `m4_0010_minimal_main.yaml`, `m5_0010_class_instantiation.yaml`, `m2_0010_compile_e003.yaml`

The index is scoped per milestone: `m1_0010` and `m2_0010` are distinct tests. Use gaps of 10 (0010, 0020, 0030…) to allow inserting new tests (e.g. 0015) without renumbering.

## File structure

Each test file has two parts:

1. **YAML front matter** — metadata between the first `---` and the next `---` (exclusive).
2. **Body** — one or more NL source blocks, each introduced by a *source separator* line and followed by the raw NL code for one logical source file.

Example:

```yaml
---
title: "Test description"
file_separator: "#NLFILE"
expected_exit_code: 0
expected_stdout: ""
---

#NLFILE test/class/ClassTest.nl
namespace test.class;
class ClassTest { ... }

#NLFILE test/class/Main.nl
namespace test.class;
class Main {
    public static int main(string[] args) { ... }
}
```

---

## Header keys

All keys are optional unless stated otherwise. Unknown keys may be ignored by a runner.

| Key | Type | Description |
|-----|------|--------------|
| `title` | string | Short human-readable description of the test. |
| `file_separator` | string | Tag used to mark the start of each NL source block (see [Source blocks](#source-blocks)). Default is `#NLFILE` if omitted. |
| `expected_exit_code` | integer | Exit code the program must return when run. If present, the test is a *run* test; the program must have exactly one `main` (see [specs.md § Entry point](specs.md#entry-point)). |
| `expected_stdout` | string | Exact stdout output expected when running the program. Empty string if no output. |
| `expected_stderr` | string | *(Optional)* Exact stderr output expected (e.g. for tests that expect compiler or runtime messages). |
| `compile_only` | boolean | If `true`, the test only checks that the sources compile successfully; no execution and no `main` required. `expected_exit_code` and `expected_stdout` are ignored. |
| `expected_compile_error` | string | If present, the test expects compilation to **fail** with this error code (e.g. `E003`). Mutually exclusive with `compile_only: true`. The runner compiles the sources and verifies the compiler reports the given error. |
| `expected_class` | string | Fully qualified class name of the module’s main class (e.g. `test.class.ClassTest`). Checked against the compiled module’s `this_class` (see [vm.md § Module format](vm.md#module-format)). When multiple modules are produced, the runner may check the module corresponding to a given path or the first/entry module; implementation-defined. |
| `expected_methods` | list of strings | Method names or signatures that must appear in the module’s method table (e.g. `["<construct>", "<destruct>"]`). Names follow VM conventions: constructors `"<construct>"`, destructors `"<destruct>"`. Entries may be simple names or full descriptors (e.g. `"(int, string) -> void"`). |
| `expected_fields` | list of strings or objects | Field names (and optionally types) that must appear in the module’s field table. Each entry is either a string (field name only) or an object with `name` and optionally `type` (type descriptor string, e.g. `"int"`, `"test.class.ClassTest"`). |
| `expected_constant_pool_contains` | list of strings | Strings or type descriptors that must appear somewhere in the module’s constant pool (e.g. class name `"test.class.ClassTest"`, type descriptor `"int"`, `"void"`). At least one module (or the designated one) must contain each listed entry. |

For a **run** test (no `compile_only`, or `compile_only: false`), the program must contain exactly one `main` method and the runner will compile all sources, run the program, and compare exit code and stdout (and optionally stderr) to the expected values.

For a **compile-only** test (`compile_only: true`), the runner only verifies that compilation succeeds; useful for testing type-checking or structure without an entry point.

For a **compile-error** test (`expected_compile_error` present), the runner compiles the sources and verifies that the compiler fails with the specified error code (e.g. `E003`). Mutually exclusive with `compile_only: true`.

The keys `expected_class`, `expected_methods`, `expected_fields`, and `expected_constant_pool_contains` are **module-structure checks**: they assert properties of the compiled module (bytecode) produced by the compiler. A runner that supports them must parse the module format (see [vm.md § Module format](vm.md#module-format)) and constant pool / descriptors to verify the expected class name, method table, field table, and constant pool contents. When the test has multiple source files, multiple modules are produced; which module(s) are checked is implementation-defined (e.g. all modules, or only the one for a given path).

---

## Source blocks

The body after the closing `---` of the front matter consists of one or more *source blocks*. Each block is:

1. **Separator line** — a line that starts with the value of `file_separator` (e.g. `#NLFILE`), followed by a space and the **logical path** of the source file.
2. **Source content** — the following lines, up to (but not including) the next separator line or end of file.

### Separator line format

- **Pattern**: `{file_separator} {path}`
- The line must begin with the exact `file_separator` string (no leading whitespace).
- One space (or more) separates the tag from the path.
- The path is the rest of the line (trailing spaces may be ignored).
- Paths use **forward slashes** and are **relative** (e.g. `test/class/ClassTest.nl`). They represent the logical path passed to the compiler; the test runner maps them to a temporary layout (e.g. namespace `test.class` → file at `test/class/ClassTest.nl`).

### Path and namespace

- The path should match the NL convention: one class (or enum) per file, file name = class name, directory hierarchy = namespace (see [specs.md § Source code files](specs.md#source-code-files)).
- Example: path `test/class/ClassTest.nl` corresponds to namespace `test.class` and class `ClassTest`.

### Block order

Blocks may appear in any order. The runner builds a virtual tree of files from the paths and compiles them together.

### Example

```text
#NLFILE test/class/ClassTest.nl
namespace test.class;
class ClassTest {
	public construct() {}
	public destruct() {}
}

#NLFILE test/class/Main.nl
namespace test.class;
use test.class.ClassTest;
class Main {
	public static int main(string[] args) {
		ClassTest a = new ClassTest();
		auto b = new ClassTest();
		return 0;
	}
}
```

The first block is the content of `test/class/ClassTest.nl`, the second is the content of `test/class/Main.nl`. There are two logical source files; the test is valid for a run (one `main` in `Main`).

---

## Summary

| Element | Rule |
|--------|------|
| File name | `m{N}_{XXXX}_{name}.yaml` — milestone, index (0010, 0020…), descriptive name. |
| Compile-error test | `expected_compile_error: "E003"` — compiler must fail with given code. |
| Front matter | YAML between first `---` and second `---`. |
| Source blocks | Introduced by a line starting with `file_separator` + space + path. |
| Path | Relative, forward slashes (e.g. `test/class/ClassTest.nl`). |
| Run test | Requires one `main` in the program; `expected_exit_code` and `expected_stdout` (and optionally `expected_stderr`) are checked. |
| Compile-only test | Set `compile_only: true`; no `main` required, no execution. |
| Module-structure keys | `expected_class`, `expected_methods`, `expected_fields`, `expected_constant_pool_contains` assert properties of the compiled module(s); see [vm.md](vm.md#module-format). |

See [specs.md](specs.md) for language rules, [compiler.md](compiler.md) for compile-time checks, and [vm.md](vm.md) for the module format and execution model.
