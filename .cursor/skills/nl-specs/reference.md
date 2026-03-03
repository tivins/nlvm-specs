# NL Specs — Reference

## Document relationships

```
specs.md (language)
    ├── compiler.md (compile-time checks)
    ├── stdlib.md (runtime API)
    └── vm.md (bytecode, execution)

tests.md (test format) → tests/*.yaml
review/coherence.md (cross-doc consistency)
```

## Key sections by doc

### specs.md
- Lexical, Types, Classes, Enums, Control flow, Operators, Exceptions, Entry point
- One class per file; namespace = folder path

### compiler.md
- Definite assignment, null safety, type checking, immutability, exception checking
- 39 error codes (E001–E038), 1 warning (W001)
- Compiler CLI: `nlc`

### stdlib.md
- Namespaces: `system`, `system.io`, `system.net`, `system.thread`, `system.time`, `system.ps`, `system.text`
- Result types: `GrepMatch`, `ProcessInfo`, `HttpResponse`, `RegexMatch`, `ProcessResult`
- Core interfaces: `Stringable`, `Cloneable`, `ValueEquatable` (in specs.md, cross-ref in stdlib)

### vm.md
- Module format, constant pool, class/method descriptors
- Instruction set (~50 opcodes)
- VM CLI: `nlvm`

### review/coherence.md
- Categories: Cross-document inconsistencies, Language omissions, Incorrect examples, VM/compiler gaps, Stdlib issues, Under-specified semantics, Editorial errors
- Format: `- [ ]` or `- [x]` with `*(fixed X.Y.Z)*` when resolved

## Test YAML keys

| Key | Use |
|-----|-----|
| `title` | Description |
| `file_separator` | Default `#NLFILE` |
| `expected_exit_code` | Run test |
| `expected_stdout` | Run test |
| `compile_only` | Compile-only test |
| `expected_class` | Module structure |
| `expected_methods` | e.g. `["<construct>", "<destruct>"]` |
| `expected_fields` | Field names/types |
| `expected_constant_pool_contains` | Constant pool entries |

## Milestones (docs/milestones.md)

1. Lexer & Parser
2. Semantic analysis
3. Bytecode emission
4. VM core
5. Objects, arrays & dispatch
6. Exceptions & closures
7. Standard library
8. Test runner & integration
9. Optimizations
