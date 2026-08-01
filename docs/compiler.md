# NL Compiler — Semantic Analyses and Compile-Time Guarantees

This document describes the **semantic analyses** and **compile-time checks** that the NL compiler must perform. It
complements [specs.md](specs.md) (language semantics) and [stdlib.md](stdlib.md) (standard library). The spec defines
*what* the language allows; this document defines *what the compiler must verify and reject*.

## Summary

* [Definite assignment analysis](#definite-assignment-analysis)
    * [Local variables](#local-variables)
    * [Variable shadowing](#variable-shadowing)
    * [Class properties](#class-properties)
* [Null safety](#null-safety)
    * [Assignment to non-nullable types](#assignment-to-non-nullable-types)
    * [Union type compatibility](#union-type-compatibility)
    * [Type narrowing (smart casts)](#type-narrowing-smart-casts)
* [Type checking](#type-checking)
    * [Auto type deduction](#auto-type-deduction)
    * [Template instantiation](#template-instantiation)
    * [Cast validation](#cast-validation)
    * [Instanceof expression](#instanceof-expression)
    * [String concatenation](#string-concatenation)
    * [Operator type compatibility](#operator-type-compatibility)
* [Immutability enforcement](#immutability-enforcement)
    * [Const methods](#const-methods)
    * [Const parameters](#const-parameters)
    * [For-each loop in const context](#for-each-loop-in-const-context)
    * [Readonly classes and properties](#readonly-classes-and-properties)
* [Exception checking](#exception-checking)
    * [Checked exception propagation](#checked-exception-propagation)
    * [Exception inheritance rules](#exception-inheritance-rules)
    * [Unreachable catch clauses](#unreachable-catch-clauses)
* [Match exhaustiveness](#match-exhaustiveness)
* [Constructor delegation](#constructor-delegation)
* [Visibility enforcement](#visibility-enforcement)
* [Inheritance modifiers (abstract, final)](#inheritance-modifiers-abstract-final)
    * [Static context restrictions](#static-context-restrictions)
    * [Duplicate definitions](#duplicate-definitions)
    * [Import name resolution](#import-name-resolution)
* [Parameter validation](#parameter-validation)
    * [Ref parameter rules](#ref-parameter-rules)
    * [Named and optional parameter rules](#named-and-optional-parameter-rules)
* [Entry point validation](#entry-point-validation)
* [Warnings](#warnings)
    * [Nodiscard](#nodiscard)
* [Reserved keywords](#reserved-keywords)
* [Default values](#default-values)
    * [Multidimensional array creation](#multidimensional-array-creation)
* [Optimizations](#optimizations)
* [Compiler invocation (nlc)](#compiler-invocation-nlc)

---

## Definite assignment analysis

The compiler tracks the initialization state of every variable and property across all control-flow paths. A variable
is *definitely assigned* at a point in the program if **every** possible execution path from its declaration to that
point includes an assignment to it. This is the same analysis as performed by Java, C#, and Kotlin compilers.

### Local variables

A local variable declared without an initializer is in an *unassigned* state. The compiler **must reject** any read
of a local variable that is not definitely assigned at the point of the read.

Control-flow constructs affect assignment status:

| Construct | Rule |
|-----------|------|
| `if/else` | The variable is definitely assigned after the `if/else` only if it is assigned in **both** branches. An `if` without `else` does not contribute. |
| `while` | The body may execute zero times, so assignments inside `while` do not make the variable definitely assigned after the loop. |
| `for` | Same as `while` for the loop body. The initializer (`init` part) contributes normally. |
| `try/catch/finally` | The variable is definitely assigned after the `try/catch` only if it is assigned in the `try` block **and** in every `catch` block. Assignments in `finally` always contribute. |
| `switch/match` | The variable is definitely assigned after the `switch` only if **every** `case` (including `default`) assigns it, and the switch is exhaustive. |

**Terminal statements.** The following statements unconditionally end the current control-flow path; any code following them in the same block is unreachable:

- `return` (with or without a value)
- `throw`
- `system.ps.Process.exit(code)` (see [stdlib.md § system.ps](stdlib.md#systemps))

In a method with a non-void return type, a path that ends with `throw` or `Process.exit()` satisfies the requirement for a return value (no missing-return error).

**Error:** `E001 — Variable '%s' may not have been initialized`

See: [specs.md § Definite assignment](specs.md#null-initialization-and-default-values)

### Variable shadowing

The compiler implements **variable shadowing** as defined in [specs.md § Variable shadowing](specs.md#variable-shadowing). A local variable or parameter may shadow a class field (the field remains accessible via `this.name`). A variable in an inner block may shadow a variable in an enclosing block. Shadowing is allowed; no error is emitted.

### Class properties

Class properties follow different rules depending on their type:

| Property type | Rule |
|---------------|------|
| Scalar (`int`, `float`, `bool`, `byte`) | Automatically initialized to default value (`0`, `0.0`, `false`, `0`). No check needed. |
| `string` | Automatically initialized to `""`. No check needed. |
| Nullable reference (`T\|null`) | Automatically initialized to `null`. No check needed. |
| Non-nullable reference (`T` where T is a class) | **Must** be assigned at declaration site or in **every** path of every `construct` overload. |

For non-nullable reference properties, the compiler must analyze all constructor bodies and verify that every code
path through the constructor assigns the property. If any path exits the constructor without assigning the property,
the compiler must reject the code.

**Error:** `E002 — Property '%s' of non-nullable type '%s' is not initialized in constructor`

---

## Null safety

### Assignment to non-nullable types

The compiler **must reject** any assignment of `null` to a variable whose declared type does not include `null` in a
union. This applies to local variables, parameters, properties, and return values.

```
string name = null;           // E003
string|null name = null;      // OK
```

**Error:** `E003 — Cannot assign 'null' to type '%s' (type does not include null)`

### Union type compatibility

When assigning a value to a union type, the compiler verifies that the value's static type is one of the union's
constituent types, or a subtype thereof per the [implicit conversions](specs.md#type-conversions-and-casting) rules
(a subclass, or a class implementing an interface constituent). Conversely, using a union-typed value in a context
that expects a specific type requires narrowing (e.g. via an `if` check, `match`, or null comparison).

```
string|int value = 42;       // OK — int is in the union
string|int value = true;     // E004 — bool is not in the union

interface Animal {}
class Dog implements Animal {}
Animal|null pet = new Dog(); // OK — Dog implements Animal, a union constituent
```

**Error:** `E004 — Type '%s' is not assignable to '%s'`

### Type narrowing (smart casts)

The compiler performs **flow-sensitive type narrowing** on union-typed values: after a type test, the value's
static type is refined within the region where the test is known to hold (as in Kotlin's smart casts or
TypeScript's narrowing). Narrowing applies to **local variables and parameters only**.

**Narrowing conditions:**

| Test | Effect |
|------|--------|
| `x != null` (or `x == null` with `else`) | In the branch where the test holds, `x: T\|null` is narrowed to `T`. In the other branch, `x` is narrowed to `null`. |
| `x instanceof C` | In the `true` branch, `x` is narrowed to `C` (and `null` is removed — `instanceof` is `false` for `null`). |
| Early exit: `if (x == null) { return; }` (or `throw`, `break`, `continue`, `Process.exit`) | After the `if`, `x` is narrowed to `T` (the null path cannot reach that point). |
| `&&` chains: `x != null && x.length() > 0` | The narrowing from the left operand applies within the right operand. |
| `\|\|` chains: `x == null \|\| x.length() == 0` | The negation of the left operand applies within the right operand (in the example, `x` is non-null in `x.length() == 0`). |
| Ternary condition | Same as `if`/`else`: each branch sees the narrowing implied by the condition. |

For general unions (`A|B|null`), each test removes or selects constituents: after `if (v instanceof A)`, `v` is
`A` in the branch; after `if (v == null) return;`, `v` is `A|B`.

**Invalidation rules** — narrowing is discarded when the compiler can no longer guarantee the tested fact:

- An **assignment** to the variable inside the narrowed region resets its type to the declared type (then
  re-narrows from subsequent tests).
- A variable **captured by a closure that mutates it** is never narrowed (the closure may run between the test
  and the use).
- **Properties are not narrowed** (`this.x != null` does not narrow `this.x` — another method or thread may
  reassign it between the test and the use). Copy the property into a local variable first:

```nl
string|null n = this.name;   // copy to a local
if (n != null) {
    system.Out.print(n.length());   // OK — n narrowed to string
}
// if (this.name != null) { this.name.length(); }  // E004 — properties are not narrowed
```

Uses of a union-typed value in a context requiring a specific type, without sufficient narrowing, are rejected
with **E004**.

---

## Type checking

### Auto type deduction

When a variable is declared with `auto`, the compiler deduces the type from the initializer expression. The deduced
type is fixed at compile-time and the variable behaves as if it were declared with that explicit type.

Rules:
- `auto` **requires** an initializer. A declaration `auto x;` without assignment is a compile-time error.
- The deduced type is the static type of the initializer expression.
- `auto` is valid for local variables, loop variables (`for (auto item : collection)` or `for (const auto item : collection)`; see [specs.md § Loops](specs.md#loops)), and variable declarations with `new`, function calls, or expressions.

**Error:** `E005 — Cannot use 'auto' without an initializer`

### Template instantiation

Templates are instantiated at compile-time. When a template class or method is used with concrete type arguments,
the compiler generates a specialized version with those types substituted and then type-checks the resulting code.

The compiler must verify:
- All operations used on the type parameter are valid for the concrete type (e.g. `a > b` requires that the type
  supports `operator>`).
- **Bounded type parameters:** When a type parameter is declared with `extends Bound` (see [specs.md § Bounded type parameters](specs.md#bounded-type-parameters)), the concrete type argument must be a subtype of the bound (implement the interface or extend the class). This is checked at instantiation time, before structural verification.
- **Unbounded parameters:** When no bound is specified, template arguments are checked structurally: all operations used on the type parameter must be valid for the concrete type.

**Error:** `E006 — Type '%s' does not support operator '%s' (required by template '%s')`  
**Error:** `E037 — Type '%s' does not satisfy bound '%s' (required by template '%s')`

### Cast validation

Cast rules are defined in the language specification: [Type conversions and casting](specs.md#type-conversions-and-casting). The cast operator `(T) expr` is validated at compile-time. The compiler accepts a cast when:

| Source → Target | Rule |
|-----------------|------|
| Primitive → same primitive | Identity, always valid. |
| Numeric widening (`byte` → `int`, `int` → `float`) | Implicit, no cast needed. |
| Numeric narrowing (`float` → `int`, `int` → `byte`) | Explicit cast required, valid. |
| Any → `string` | Valid if source is a primitive or implements `Stringable`. Otherwise compile-time error. |
| Class → superclass | Always valid (upcast). |
| Superclass → subclass | Valid at compile-time, checked at runtime (throws `InvalidCastException` on failure). |
| Unrelated class → class | Compile-time error. |
| `T|null` → `T` | Not allowed (reducing nullability by cast alone). |

**Error:** `E007 — Cannot cast '%s' to '%s'`

### Instanceof expression

The `instanceof` operator (`expr instanceof ClassName`) performs a runtime type test. The compiler validates:

- **Left operand:** Must be a reference type (class, interface, array, `string`, or `T|null`). Primitives (`int`, `float`, `bool`, `byte`) are rejected.
- **Right operand:** Must be a class or interface type (not a primitive, not `void`).

Invalid operand types produce a compile-time error.

See [specs.md § Other operators](specs.md#other-operators) and [vm.md § Object operations](vm.md#object-operations).

### String concatenation

The `+` operator performs string concatenation when at least one operand is `string`. The compiler must verify that
the non-string operand is either:
- A primitive type (`int`, `float`, `bool`, `byte`) — implicit conversion to string.
- A reference type that implements `Stringable` — calls `toString()`.

Otherwise the expression is a compile-time error.

**Error:** `E008 — Cannot concatenate 'string' with type '%s' (type does not implement Stringable)`

### Operator type compatibility

For every binary operator expression `a op b`, the compiler verifies that:
1. The left operand's type defines `operator op` accepting the right operand's type, **or**
2. Both operands are primitives and the operation is a built-in operator for those types.

For unary operators (`!`, `-`, `++`, `--`), the operand type must be a primitive that supports the operator, or a
class that overloads it.

**Error:** `E009 — Operator '%s' is not defined for types '%s' and '%s'`

---

## Immutability enforcement

### Const methods

A method declared `const` (after the parameter list) cannot modify any property of `this`. Inside a const method,
the compiler must reject:
- Any assignment to `this.property`.
- Any call to a non-const method on `this`.

**Error:** `E010 — Cannot modify property '%s' in a const method`
**Error:** `E011 — Cannot call non-const method '%s' in a const method`

When a class implements an interface, a method that implements an interface method declared as `const` must itself be declared `const`. For example, [Stringable](specs.md#stringable-interface) declares `string toString() const`; any implementing class must provide `string toString() const` (not a non-const `toString()`).

**Error:** `E044 — Method '%s' implementing interface '%s' must be declared const`

### Const parameters

A parameter declared `const` cannot be reassigned or mutated inside the method body. For object types, only
`const` methods may be called on the parameter.

**Error:** `E012 — Cannot modify const variable '%s'`

### Const local variables

A local variable declared `const` (e.g. `const int x = 42;`) cannot be reassigned after its initial assignment.
For object types, only `const` methods may be called on the variable.

**Error:** `E012 — Cannot modify const variable '%s'` (same code for const parameters and const locals)

### For-each loop in const context

When a for-each loop iterates over a collection that is read-only in the current scope, the loop variable is implicitly non-modifiable (see [specs.md § Loops](specs.md#loops)). This applies when:

1. The loop is inside a **const method** and the collection is a property of `this` (e.g. `for (auto item : this.items)`).
2. The collection is a **const parameter** or **const ref parameter** (e.g. `void f(const int[] arr) { for (auto x : arr) { ... } }`).

In these cases, the compiler must reject any assignment to the loop variable and any call to a non-const method on it.

**Error:** `E039 — Cannot modify loop variable '%s' — implicitly const when iterating over read-only collection`

### Readonly classes and properties

- A `readonly` **class**: after construction, **no** property of any instance can be modified. The compiler must
  reject any assignment to a property outside of `construct`.
- A `readonly` **property**: only that specific property is immutable after construction.

In both cases, assignment is allowed inside `construct` (including via `this.property = ...`).

**Error:** `E013 — Cannot modify property '%s' of readonly class '%s'`
**Error:** `E014 — Cannot modify readonly property '%s'`

---

## Exception checking

### Checked exception propagation

The compiler enforces that every checked exception (any exception that is not a subclass of `RuntimeException`) is
either:
1. Caught in a surrounding `try/catch` block with a compatible catch clause, **or**
2. Declared in the enclosing method's `throws` clause.

This applies to:
- Direct `throw` statements.
- Calls to methods that declare `throws`.
- Constructor calls (`new`) when the constructor declares `throws`.

Runtime exceptions (`RuntimeException` and subclasses) are exempt: they do not require `throws` declarations and
can propagate freely.

**Documentation-only `throws`:** A method **may** list runtime exception types in its `throws` clause for documentation
purposes (e.g. `static int parse(string s) throws NumberFormatException`). The compiler **does not enforce** such
declarations: callers are not required to catch or re-declare them, and the compiler does not emit E015 for runtime
exceptions regardless of whether they appear in a `throws` clause. This allows API authors to signal common failure
modes without imposing try/catch on callers.

**Error:** `E015 — Unhandled checked exception '%s' — must be caught or declared in 'throws'`

### Exception inheritance rules

When a method overrides a parent method, the overriding method's `throws` clause must **cover** every checked
exception declared by the parent. Specifically:
- For each exception type `E` in the parent's `throws` clause, the child's `throws` clause must include `E` or a
  subclass of `E`.
- The child may not introduce new checked exception types that are not subtypes of exceptions declared by the parent.

Runtime exceptions listed in a `throws` clause (see [Documentation-only `throws`](#checked-exception-propagation))
are **not subject to these inheritance rules**. An overriding method may freely add or remove runtime exception
types from its `throws` clause. E016 and E017 apply only to checked exceptions.

**Error:** `E016 — Overriding method does not declare exception '%s' from parent method`
**Error:** `E017 — Overriding method declares exception '%s' not thrown by parent method`

### Unreachable catch clauses

Catch clauses of a `try` are tested in declaration order (see [specs.md § Exception handling](specs.md#exception-handling)).
The compiler must reject a catch clause that can never execute because an **earlier** clause of the same `try`
already catches the same exception type or a superclass of it. This includes two catch clauses with the exact
same exception type.

```
try { ... }
catch (Exception ex) { ... }
catch (IOException ex) { ... }   // E048 — Exception above already catches IOException
```

**Error:** `E048 — Unreachable catch clause: '%s' is already caught by earlier clause '%s'`

---

## Match exhaustiveness

A `match` expression must be exhaustive at compile time (see [specs.md § Match exhaustiveness](specs.md#match-exhaustiveness)).
The compiler must verify:

- **Enum subject:** every enum case is covered by an arm, or a `default` arm is present.
- **`bool` subject:** both `true` and `false` are covered, or a `default` arm is present.
- **`int`, `string`, and any other subject type:** a `default` arm is required.
- **Duplicate arms:** two arms with the same constant value make the second unreachable — rejected with the same
  error code.
- A `default` arm, when present, must be the **last** arm.

Because exhaustiveness is verified statically, no runtime "unmatched value" error exists.

**Error:** `E047 — Match expression is not exhaustive (missing '%s')`

---

## Constructor delegation

A constructor may delegate to another constructor of the same class (`this(...)`) or of the parent class
(`super(...)`); see [specs.md § Constructor chaining](specs.md#constructor-chaining-this). The compiler must verify:

- A constructor contains **at most one** delegation call (`this(...)` or `super(...)`, never both), and it is the
  **first statement** of the constructor body (E045).
- `this(...)` delegation chains are **acyclic**. The compiler builds the delegation graph of the class's
  constructors and rejects any cycle (E046).
- The delegation target is resolved by overload resolution on the argument types; no matching constructor is a
  regular resolution error.
- **Definite assignment:** for property initialization analysis ([§ Class properties](#class-properties)), a
  constructor that delegates with `this(...)` is credited with every property assignment guaranteed by the target
  constructor.

**Error:** `E045 — Constructor delegation call ('this' or 'super') must be the first statement`
**Error:** `E046 — Constructor delegation cycle detected in class '%s'`

---

## Visibility enforcement

The compiler enforces access rules based on visibility modifiers:

| Modifier | Accessible from |
|----------|----------------|
| `public` | Anywhere. |
| `protected` | The declaring class and its subclasses. |
| `private` | The declaring class only. |

Every class member (properties, methods, constructors, destructors) **must** have an explicit visibility modifier.
Omitting it is a compile-time error.

**Error:** `E018 — Member '%s' is not accessible from '%s' (visibility: %s)`
**Error:** `E019 — Missing visibility modifier on member '%s'`

---

## Inheritance modifiers (abstract, final)

The compiler enforces the rules for `abstract` and `final` as defined in [specs.md § Abstract classes and methods](specs.md#abstract-classes-and-methods) and [specs.md § Final classes and methods](specs.md#final-classes-and-methods).

### Abstract classes and methods

- An abstract class cannot be instantiated with `new`.
- A class that declares or inherits an abstract method without implementing it must be declared `abstract`.
- An abstract method has no body (ends with `;`).

**Error:** `E032 — Cannot instantiate abstract class '%s'`
**Error:** `E033 — Class '%s' must be declared abstract (has unimplemented abstract method '%s')`
**Error:** `E034 — Abstract method '%s' cannot have a body`

### Final classes and methods

- A final class cannot be extended.
- A final method cannot be overridden.
- `abstract` and `final` are mutually exclusive — on a class as on a method (see
  [specs.md § Final classes and methods](specs.md#final-classes-and-methods)). `readonly` combines freely with
  either (`abstract class readonly`, `final class readonly`); modifier order is
  `[abstract | final] class [readonly] Name` (see [specs.md § Readonly](specs.md#readonly)).

**Error:** `E035 — Cannot extend final class '%s'`
**Error:** `E036 — Cannot override final method '%s'`
**Error:** `E049 — Conflicting modifiers 'abstract' and 'final' on '%s'`

### Static context restrictions

Inside a `static` method, the compiler must reject:
- Use of `this` (there is no instance).
- Use of `Self` (the return-type shorthand for fluent methods relies on an instance context).
- Access to instance (non-static) properties or methods without an explicit receiver.

**Error:** `E040 — Cannot use '%s' in a static context`

### Duplicate definitions

- If a class declares two methods with the same name **and** the same parameter types (identical signature), the compiler must reject the duplicate.
- If two modules define the same fully qualified class name, the compiler (or linker) must reject the conflict.

**Signature** for duplicate detection means **method name and parameter types only**. Return type and `throws` clause do **not** distinguish methods: two methods with the same name and parameter types are duplicates (E041) even if they differ by return type or by declared exceptions.

**Error:** `E041 — Duplicate method '%s' with identical signature in class '%s'`
**Error:** `E042 — Duplicate class definition '%s'`

### Import name resolution

An import introduces a name (or alias) into the current scope. The compiler must reject any import that would create a duplicate unqualified name. A conflict occurs when:

- The imported name (or alias) matches the class being defined in the file, or
- The imported name (or alias) matches another type in the same namespace, or
- The imported name (or alias) matches another import's name or alias.

To resolve a conflict, the user must either use the `as` keyword to assign a distinct alias, or use the fully qualified name without importing. See [specs.md § Import rules](specs.md#imports).

**Error:** `E043 — Import creates duplicate symbol '%s'`

---

## Parameter validation

### Ref parameter rules

When a parameter is declared `ref`, the compiler must verify:
- The caller uses the `ref` keyword at the call site.
- The argument is a writable variable (not a literal, expression result, or const variable).
- Optional parameters cannot be `ref`.

When a parameter is declared `const ref`, the argument must still be a variable (to bind the reference), but the
method body cannot modify it.

**Error:** `E020 — Argument for 'ref' parameter '%s' must be a variable`
**Error:** `E021 — Missing 'ref' keyword at call site for parameter '%s'`
**Error:** `E022 — Optional parameters cannot be declared 'ref'`

### Named and optional parameter rules

The compiler validates call sites with named parameters:
- All required parameters (those without default values) must be provided, either positionally or by name.
- Positional arguments must precede all named arguments.
- A parameter cannot be provided both positionally and by name.
- Default values for optional parameters must be compile-time constants.

**Error:** `E023 — Required parameter '%s' not provided`
**Error:** `E024 — Positional argument after named argument`
**Error:** `E025 — Parameter '%s' provided both positionally and by name`
**Error:** `E026 — Default value for parameter '%s' must be a compile-time constant`

---

## Entry point validation

The compiler verifies that the program contains exactly one `main` method with the signature:

```
public static int main(string[] args)
```

- The method must be `public static`.
- The return type must be `int`.
- The parameter list must be `(string[])`.
- Exactly **one** such method must exist across the entire program. Zero or more than one is an error.
- This requirement does not apply to library projects (no entry point needed).

**Error:** `E027 — No 'main' method found`
**Error:** `E028 — Multiple 'main' methods found`
**Error:** `E029 — 'main' method has incorrect signature (expected: public static int main(string[]))`

---

## Warnings

### Nodiscard

When a method or function is marked `nodiscard`, calling it and discarding the return value produces a
**warning** (not an error). The compiler should emit a diagnostic but not reject the program.

**Warning:** `W001 — Return value of nodiscard method '%s' is discarded`

---

## Reserved keywords

The following identifiers are reserved and cannot be used as variable names, method names, class names, or any other
identifier:

- All keywords listed in [specs.md § Keywords](specs.md#keywords).
- `undefined` — reserved keyword that does not participate in the type system.

**Error:** `E030 — '%s' is a reserved keyword and cannot be used as an identifier`

---

## Default values

The compiler applies default values in two contexts:

### Array creation with fixed size

`new T[n]` creates an array of `n` elements, each initialized to the default value for type `T`:

| Type | Default |
|------|---------|
| `int`, `byte` | `0` |
| `float` | `0.0` |
| `bool` | `false` |
| `string` | `""` |
| Reference type (`T\|null`) | `null` |
| Non-nullable reference type | **Not allowed** — `new T[n]` where `T` is a non-nullable reference type is a compile-time error (there is no valid default). |

**Error:** `E031 — Cannot create array of non-nullable type '%s' with fixed size (no default value)`

### Multidimensional array creation

`new T[n₁][n₂]…[nₖ]` is syntactic sugar. The compiler desugars it into nested allocations:

1. Evaluate all dimension expressions `n₁, n₂, …, nₖ` left-to-right and store in temporaries.
2. Emit `NEW_ARRAY` for the outermost array (element type = `T[]…[]` with k−1 brackets, size = `n₁`).
3. For each index `i` in `[0, n₁)`, emit a loop that allocates the next-level array (`NEW_ARRAY` with element type = `T[]…[]` with k−2 brackets, size = `n₂`) and stores it with `ARRAY_STORE`. Repeat recursively for each remaining dimension.

**Partial dimensions** (`new T[n₁][]`): the compiler only allocates the outermost array; inner elements are `null`. The element type of the outer array is the inner array type as nullable (e.g. `int[]|null`).

Dimension sizes may only be omitted as a **contiguous suffix** from the right: `new int[3][][]` is valid, `new int[][3][]` is not.

**Error:** `E038 — Non-first dimension size omitted in middle position in '%s'`

`E031` still applies at each level: `new MyClass[n]` is rejected if `MyClass` is non-nullable and has no default value.

### Class property defaults

Properties declared without an initializer receive the default value for their type (same table as above).
Non-nullable reference properties have no default and must be initialized — see
[Definite assignment § Class properties](#class-properties).

---

## Error code summary

| Code | Category | Description |
|------|----------|-------------|
| E001 | Definite assignment | Variable may not have been initialized |
| E002 | Definite assignment | Non-nullable property not initialized in constructor |
| E003 | Null safety | Cannot assign null to non-nullable type |
| E004 | Type checking | Type not assignable to target type |
| E005 | Auto deduction | Auto without initializer |
| E006 | Templates | Type does not support required operator |
| E037 | Templates | Type does not satisfy template bound |
| E007 | Cast | Invalid cast |
| E008 | String concatenation | Non-Stringable type in concatenation |
| E009 | Operators | Operator not defined for types |
| E010 | Const | Property modification in const method |
| E011 | Const | Non-const method call in const method |
| E012 | Const | Modification of const parameter or variable |
| E039 | Const | Loop variable implicitly const (for-each over read-only collection) |
| E013 | Readonly | Property modification on readonly class |
| E014 | Readonly | Modification of readonly property |
| E015 | Exceptions | Unhandled checked exception |
| E016 | Exceptions | Missing exception in overriding method |
| E017 | Exceptions | New exception in overriding method |
| E048 | Exceptions | Unreachable catch clause |
| E047 | Match | Match expression not exhaustive |
| E045 | Constructors | Delegation call not first statement |
| E046 | Constructors | Constructor delegation cycle |
| E018 | Visibility | Member not accessible |
| E019 | Visibility | Missing visibility modifier |
| E020 | Ref params | Ref argument must be a variable |
| E021 | Ref params | Missing ref keyword at call site |
| E022 | Ref params | Optional parameter cannot be ref |
| E023 | Named params | Required parameter not provided |
| E024 | Named params | Positional argument after named |
| E025 | Named params | Parameter provided both ways |
| E026 | Named params | Default value not a compile-time constant |
| E027 | Entry point | No main method |
| E028 | Entry point | Multiple main methods |
| E029 | Entry point | Incorrect main signature |
| E030 | Keywords | Reserved keyword used as identifier |
| E031 | Arrays | Fixed-size array of non-nullable type |
| E038 | Arrays | Non-first dimension size omitted in middle position |
| E032 | Abstract/Final | Cannot instantiate abstract class |
| E033 | Abstract/Final | Class must be abstract (unimplemented abstract method) |
| E034 | Abstract/Final | Abstract method cannot have body |
| E035 | Abstract/Final | Cannot extend final class |
| E036 | Abstract/Final | Cannot override final method |
| E049 | Abstract/Final | Conflicting `abstract` and `final` modifiers |
| E040 | Static context | Cannot use `this`, `Self`, or instance member in static method |
| E041 | Duplicates | Duplicate method with identical signature |
| E042 | Duplicates | Duplicate class definition |
| E043 | Imports | Import creates duplicate symbol |
| E044 | Const | Method implementing interface must be const |
| W001 | Warning | Nodiscard return value discarded |

---

## Optimizations

The compiler **may** apply optional optimizations such as constant folding, dead code elimination, devirtualization,
inlining, and incremental compilation. All optimizations must preserve observable semantics. See [optimizations.md](optimizations.md) for the
full optimization contract (compiler and VM).

Which optimizations run is selected by `-O<n>` (see [Options](#options)). `-O0` — no optimization at all — **must**
be supported, so that the same program can be built both ways and the two builds compared
([optimizations.md § Testing](optimizations.md#testing)). Which passes any level above `0` applies remains free
(principle 3, *implementation freedom*).

---

## Compiler invocation (nlc)

The compiler is invoked as:

    nlc [options] <sources...>

### Arguments

| Argument | Description |
|----------|-------------|
| `<sources...>` | One or more NL source files (`.nl`). Each file is compiled to one module (see [vm.md § Module format](vm.md#module-format)). Order may matter for entry/module resolution; the first file or a designated entry is used as the program entry when producing an executable bundle. |

### Options

| Option | Description |
|--------|-------------|
| `-o <dir>`, `--output <dir>` | Output directory for emitted modules (`.nlm`). Default: current directory or implementation-defined. |
| `--entry <path>` | *(Optional)* Path of the source file that contains the program entry point (`main`). If omitted, the implementation may use the first file or require exactly one module with `main`. |
| `-c` | Compile only; emit modules and do not link or produce a runnable bundle. |
| `--version` | Print compiler version and exit. |
| `-h`, `--help` | Print usage and exit. |
| `-Werror` | Treat all warnings as errors. |
| `-v`, `--verbose` | Verbose diagnostics (implementation-defined). |
| `-O<n>` | Optimization level. `-O0` applies **no** optimization and **must** be a supported configuration; `-O1` applies an implementation-chosen set from [optimizations.md § Compiler optimizations](optimizations.md#compiler-optimizations); levels above `1` are implementation-defined. The default level when the option is absent is implementation-defined. The selected level is recorded in every emitted module (`opt_level`, see [vm.md § Module format](vm.md#module-format)). See [optimizations.md § Optimization levels](optimizations.md#optimization-levels). |
| `--incremental`, `-i` | *(Optional)* Enable incremental compilation. Cache compiled modules and recompile only changed files and their dependents. See [optimizations.md § Compiler optimizations](optimizations.md#compiler-optimizations). |

`-O<n>` governs **code transformations** only. Incremental compilation is a build strategy, not a transformation of
the emitted code, and is controlled solely by `--incremental`: `-O0 --incremental` is a valid combination and must
produce the same modules as `-O0` alone.

### Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success; all sources compiled. |
| Non-zero | Compilation failed (syntax error, semantic error per [Error code summary](#error-code-summary), or I/O error). |

### Conventions

- Source paths follow the one-class-per-file convention; directory structure reflects namespace (see [specs.md § Source code files](specs.md#source-code-files)).
- Output module names or paths are derived from the fully qualified class name (e.g. `com.example.App` → `com/example/App.nlm` or implementation-defined).
