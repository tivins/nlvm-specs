---
name: nl-specs
description: Guides the agent when writing or editing NL language specifications. Use when working on specs.md, compiler.md, stdlib.md, vm.md, coherence tracking, or YAML tests in tests/. Covers cross-document consistency, test format (docs/tests.md), and milestone references.
---

# NL Language Specifications

Assists with writing, editing, and maintaining the NL language specification documents. NL is a statically-typed OOP language; the repo contains specs only (no compiler/VM implementation).

## Document structure

| Doc | Purpose |
|-----|---------|
| `docs/specs.md` | Language syntax and semantics |
| `docs/compiler.md` | Semantic analyses, 39 error codes (E001–E038), 1 warning (W001) |
| `docs/stdlib.md` | Standard library API (`system`, `system.io`, etc.) |
| `docs/vm.md` | Bytecode format, instruction set, execution model |
| `docs/tests.md` | Test file format (YAML, source blocks) |
| `docs/optimizations.md` | Compiler and VM optimization contract |
| `docs/milestones.md` | Implementation roadmap (9 phases) |
| `review/coherence.md` | Inconsistencies, omissions, gaps across all docs |

See [reference.md](reference.md) for conventions and cross-references.

---

## Workflows

### Writing or editing a spec section

1. Identify which doc(s) are affected (specs ↔ compiler ↔ stdlib ↔ vm).
2. Add or update content following existing style (examples in NL, anchors for links).
3. Add cross-references to related sections in other docs.
4. If adding a language feature: consider compiler checks (compiler.md), VM representation (vm.md), stdlib impact (stdlib.md).
5. Update CHANGELOG.md with the change.

### Coherence check

1. Read [review/coherence.md](review/coherence.md) for known issues.
2. When fixing an item: check the box, add `*(fixed X.Y.Z)*` with version.
3. When adding new content: verify consistency with other docs; add an entry to coherence.md if you find a gap.
4. Resolved items may be moved to [review/archives/](review/archives/) when a coherence pass is closed.

### Creating or editing a YAML test

1. Follow format in [docs/tests.md](docs/tests.md).
2. Front matter: `title`, `file_separator` (default `#NLFILE`), `expected_exit_code` / `expected_stdout` for run tests, or `compile_only: true` for compile-only.
3. Source blocks: `#NLFILE path/to/Class.nl` then NL code. Path = logical path (forward slashes); path matches namespace (e.g. `test/class/Main.nl` → `namespace test.class`).
4. Module-structure checks: `expected_class`, `expected_methods`, `expected_fields`, `expected_constant_pool_contains` when applicable.

Example:

```yaml
---
title: "Test description"
file_separator: "#NLFILE"
expected_exit_code: 0
expected_stdout: ""
---

#NLFILE test/example/Main.nl
namespace test.example;
class Main {
    public static int main(string[] args) { return 0; }
}
```

### Adding a new feature (full flow)

1. **specs.md** — define syntax and semantics.
2. **compiler.md** — add error codes if new checks are needed.
3. **stdlib.md** — add API if the feature affects the standard library.
4. **vm.md** — add opcodes, descriptors, or compilation strategies if needed.
5. **tests/** — add YAML test(s).
6. **review/coherence.md** — add entry if any ambiguity or gap remains.

---

## Conventions

- **Examples**: Use valid NL code; ensure types match (e.g. `MyObject|null` for null, not `MyObject`).
- **Links**: Use relative anchors like `[specs.md § Loops](specs.md#loops)`.
- **Versioning**: Update CHANGELOG.md with semantic version (major.minor.patch).
- **Coherence**: Always reference `review/coherence.md` for tracking; never reference `docs_internal/`.

---

## Quick reference

- **Error codes**: E001–E038, W001 — see compiler.md.
- **Test format**: YAML front matter + `#NLFILE` blocks — see docs/tests.md.
- **Namespace**: One class per file; path = directory hierarchy.
