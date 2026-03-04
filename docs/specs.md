# NL Language Specification

NL is a statically-typed, object-oriented programming language that combines the best features from multiple modern
languages. It draws inspiration from C++ for its powerful type system and templates, adopts Java's clean syntax and file
organization, incorporates modern PHP 8 features like match expressions and typed enums, and adopts Kotlin-style null
safety with compile-time initialization guarantees for precise state representation.

The language emphasizes type safety through compile-time type checking while providing convenience features like
automatic type deduction with the `auto` keyword. It offers comprehensive immutability guarantees through both `const` (
for methods, parameters, and local variables) and `readonly` (for classes and properties), enabling developers to write safer and more
maintainable code.

NL supports modern programming paradigms including classes with inheritance and interfaces, template-based generics,
operator overloading, and exception handling. The language is designed to be expressive yet safe, allowing developers to
write concise code without sacrificing type safety or runtime guarantees.

## Summary

* [Lexical Elements](#lexical-elements)
    * [Source code files](#source-code-files)
    * [Imports](#imports)
    * [Keywords](#keywords)
    * [Identifiers](#identifiers)
* [Types](#types)
    * [Native types](#native-types)
    * [Arrays](#arrays)
    * [Null, initialization, and default values](#null-initialization-and-default-values)
    * [Union types and explicit nullable](#union-types-and-explicit-nullable)
    * [Type conversions and casting](#type-conversions-and-casting)
    * [Auto type](#auto-type)
    * [Typedef](#typedef)
* [Classes](#classes)
    * [Basic class](#basic-class)
    * [Visibility](#visibility)
    * [Self and type keywords](#self-and-type-keywords)
    * [Super keyword](#super-keyword)
    * [Constructors and Destructors](#constructors-and-destructors)
    * [Class methods](#class-methods)
    * [Parameter passing semantics](#parameter-passing-semantics)
    * [Named parameters](#named-parameters)
    * [Fluent methods](#fluent-methods)
    * [Readonly](#readonly)
    * [Nodiscard](#nodiscard)
    * [Template class](#template-class)
        * [Bounded type parameters](#bounded-type-parameters)
    * [Template methods](#template-methods)
    * [Extends, Implements](#extends-implements)
    * [Abstract classes and methods](#abstract-classes-and-methods)
    * [Final classes and methods](#final-classes-and-methods)
    * [Virtual method dispatch](#virtual-method-dispatch)
    * [Cloneable interface](#cloneable-interface)
    * [ValueEquatable interface](#valueequatable-interface)
* [Enums](#enums)
    * [Basic enums](#basic-enums)
    * [Typed enums](#typed-enums-string-or-int)
    * [Enum methods](#enums-methods)
* [Control Structures](#control-structures)
    * [Conditionals](#conditionals)
    * [Loops](#loops)
    * [Switch/Match](#switchmatch)
* [Anonymous Functions](#anonymous-functions)
    * [Typedef advanced usage](#typedef-advanced-usage)
* [Operators](#operators)
    * [String concatenation](#string-concatenation)
    * [Conditional operators](#conditional-operators)
* [Operator Overloading](#operator-overloading)
* [Exceptions](#exceptions)
    * [Exception class hierarchy](#exception-class-hierarchy)
    * [Declaring exceptions](#declaring-exceptions)
    * [Exception handling](#exception-handling)
    * [Constructors with exceptions](#constructors-with-exceptions)
    * [Exception inheritance rules](#exception-inheritance-rules)
* [Standard library](stdlib.md)
* [Entry point](#entry-point)
* [Planned](#planned)

## Lexical Elements

### Source code files

Like Java, NL requires that the definition of an object (class, enumeration, etc.) be in a file of the same name. The
namespace must correspond to the folder hierarchy.

```nl
// file: <root>/src/com/example/myProject/MyClass.nl
namespace com.example.myProject;
class MyClass {}
```

### Imports

NL uses the `use` keyword to import class definitions from other namespaces. This allows you to reference classes from other packages without using their fully qualified names. The `as` keyword can be used to provide an alias when importing, which is useful when there are naming conflicts or when you want to use a shorter name.

#### Basic usage

```nl
namespace com.example.io;

use com.example.utils.FileHelper;
use com.example.data.User;

class FileProcessor {
    public void process() {
        auto helper = new FileHelper();
        auto user = new User();
    }
}
```

#### Using aliases

When importing a class, you can use the `as` keyword to provide an alias. This is particularly useful when:
- There are naming conflicts (multiple classes with the same name from different packages)
- You want to use a shorter or more descriptive name
- You want to improve code readability

```nl
namespace com.example.io;

use com.example.utils.FileHelper;
use com.otherPackage.FileHelper as OtherFileHelper;
use com.example.data.User as UserData;

class FileProcessor {
    public void process() {
        auto helper = new FileHelper(); // Uses com.example.utils.FileHelper
        auto otherHelper = new OtherFileHelper(); // Uses com.otherPackage.FileHelper
        auto user = new UserData(); // Uses com.example.data.User
    }
}
```

#### Import rules

- Imports must be declared at the top of the file, after the `namespace` declaration
- Multiple imports can be declared, one per line
- Imported classes can be used without their fully qualified names
- If a class is not imported, you must use its fully qualified name (e.g., `com.example.utils.FileHelper`)
- The `as` keyword is optional; if omitted, the class is imported with its original name
- Import aliases must be unique within the current namespace

```nl
namespace com.example.app;

use com.example.utils.Logger;
use com.example.data.User;
use com.example.data.Product as Item;

class Application {
    public void run() {
        auto logger = new Logger(); // Uses imported class
        auto user = new User(); // Uses imported class
        auto item = new Item(); // Uses imported class with alias
        
        // Fully qualified name still works
        auto directUser = new com.example.data.User();
    }
}
```

### Keywords

|                          |                                                                                                                      |
|--------------------------|----------------------------------------------------------------------------------------------------------------------|
| Control flow             | [`if`][1], [`else`][1], [`while`][2], [`for`][2], [`break`][2], [`continue`][2], [`switch`][3], [`case`][3], [`match`][3]             |
| Types and literals       | [`auto`][4], [`const`][5], [`ref`][23], [`void`][6], `true`, `false`, [`null`][7], `return`                                      |
| Reserved                 | `undefined`                                                                                                          |
| Object-oriented          | [`class`][18], `interface`, [`namespace`][8], [`extends`][9], [`implements`][9], [`super`][21], [`Self`][10], [`type`][10], [`use`][22], [`as`][22] |
| Visibility and modifiers | [`private`][19], [`public`][19], [`protected`][19], [`static`][5], [`final`][24], [`abstract`][25], [`readonly`][11], [`nodiscard`][20]       |
| Templates and generics   | [`template`][12], [`operator`][13]                                                                                   |
| Exceptions               | [`try`][14], [`catch`][14], [`finally`][14], [`throw`][15], [`throws`][15]                                           |
| Lifecycle                | [`construct`][16], [`destruct`][16], `new`                                                                          |
| Other                    | [`enum`][17], [`typedef`](#typedef)                                                                                              |

### Identifiers

- **Starting characters**: Letters (a-z, A-Z), underscore (`_`), or any valid UTF-8 character
- **Subsequent characters**: Letters, digits (0-9), underscore (`_`), or any valid UTF-8 character

Valid examples:

```nl
variable
ret_val
retVal
RetVal
var1
var_1
_var
var_
élément
```

Invalid examples

```nl
1var // Error : Start with a number
my-var // Error : Contains '-'
my var // Error : Contains ' '
```

## Types

### Native types

* `string`, `int`, `float`, `bool` (`true`, `false`), `byte`, `null`
* `void` - special type for methods that do not return a value
* Arrays uses `[]`: `string[]`, `int[]`, `byte[]`, `customObject[]`

NL does not provide byte literal syntax. To obtain a byte value, use an explicit cast: `(byte) intExpr`. The expression must evaluate to an integer; values outside 0–255 are treated as in [int → byte conversion](#type-conversions-and-casting) (low-order bits; overflow implementation-defined).

A **character** is represented as a `string` of length 1 (e.g. via `system.String.charAt`). A dedicated `char` type may be added to the spec in a future version; see [Planned](#planned).

See [Arrays](#arrays) for creation, access, and built-in methods.

### Arrays

Arrays are fixed-size contiguous sequences of elements of type `T`. The type is written `T[]` (e.g. `int[]`, `string[]`, `byte[]`). Arrays support index access, a fixed set of methods, and work with the [for-each loop](#loops).

#### Creation

Two forms are supported:

- **Initializer list**: `new T[]{ element1, element2, ... }` — the array size is the number of elements; values are copied in order. An empty array is `new T[]{}`.
- **Fixed size**: `new T[n]` — creates an array of length `n` with default values (e.g. `0` for `int`, `false` for `bool`, `null` for reference types).

```nl
auto a = new int[]{1, 2, 3, 4, 5};
auto b = new int[256];
auto empty = new string[]{};
```

Arrays are fixed-size after creation; for dynamic growth or shrink, use `system.List<T>` (see [stdlib](stdlib.md)).

#### Multidimensional arrays

A multidimensional array is an array of arrays. The type `T[][]` is equivalent to `(T[])[]` — an array whose elements are themselves arrays of `T`. Deeper nesting (`T[][][]`, etc.) follows the same principle.

**Fixed-size creation:**

`new T[n₁][n₂]…[nₖ]` creates a k-dimensional array. The outermost array has `n₁` elements, each of which is an array of `n₂` elements, and so on. All leaf elements are initialized to the [default value](#null-initialization-and-default-values) for `T`.

```nl
int[][] matrix = new int[3][4];      // 3 rows of 4 ints, all 0
string[][] grid = new string[2][2];  // 2×2 grid, all ""
```

**Partial dimension:** only the first dimension is required. Trailing dimensions may be omitted, producing an array of `null` references that must be assigned before use:

```nl
int[][] ragged = new int[3][];
ragged[0] = new int[]{1, 2};
ragged[1] = new int[]{3, 4, 5};
ragged[2] = new int[]{6};
```

When trailing dimensions are omitted, the element type of the outer array is nullable (`int[]|null` in the example above) until each element is assigned.

**Initializer list:**

```nl
int[][] m = new int[][]{
    new int[]{1, 2, 3},
    new int[]{4, 5, 6}
};
```

Access uses chained indexing: `matrix[row][col]`.

Only the first dimension size may be omitted in a middle position — sizes can only be left out as a **contiguous suffix** from the right. For example, `new int[3][][]` is valid (allocates only the outermost array) but `new int[][3][]` is a compile-time error (see [compiler.md § Multidimensional array creation](compiler.md#multidimensional-array-creation)).

#### Access (get/set)

The subscript operator `[i]` is used to read or write the element at index `i` (zero-based). Accessing an index outside the valid range `[0, length()-1]` throws **`IndexOutOfBoundsException`** (a runtime exception).

```nl
int[] arr = new int[]{10, 20, 30};
int x = arr[0];   // 10
arr[1] = 21;     // write
// arr[10] = 0;  // may throw IndexOutOfBoundsException
```

#### Built-in methods

Arrays provide the following methods. Callbacks use [anonymous function](#anonymous-functions) syntax.

| Method | Signature / behavior | Description |
|--------|----------------------|-------------|
| `length` | `int length()` | Returns the number of elements (size of the array). |
| `slice` | `T[] slice(int start, int end)` | Returns a **new** array containing elements from index `start` (inclusive) to `end` (exclusive). Bounds are clamped to the array range. |
| `map` | `U[] map((T value) => U f)` | Returns a new array of the same length where each element is `f(element)`. |
| `filter` | `T[] filter((T value) => bool predicate)` | Returns a new array containing only elements for which `predicate(value)` is `true`. |
| `forEach` | `void forEach((T value) => void f)` | Invokes `f` for each element in order. |
| `sort` | `void sort((T a, T b) => int compare)` | Sorts the array in place. `compare(a, b)` returns negative if `a` &lt; `b`, zero if equal, positive if `a` &gt; `b`. |
| `find` | `T\|null find((T value) => bool predicate)` | Returns the first element for which `predicate(value)` is `true`, or `null` if none. |

```nl
int[] numbers = new int[]{1, 2, 3, 4, 5};
int len = numbers.length();                    // 5
int[] sub = numbers.slice(1, 4);               // [2, 3, 4]
int[] doubled = numbers.map((int n) => n * 2); // [2, 4, 6, 8, 10]
int[] evens = numbers.filter((int n) => n % 2 == 0); // [2, 4]
numbers.forEach((int n) => { system.Out.print(n); });
numbers.sort((int a, int b) => a - b);
auto found = numbers.find((int n) => n > 3);   // 4 or null
```

For dynamic operations (add/remove at front or back), use **`system.List<T>`** (see [stdlib](stdlib.md)).

### Null, initialization, and default values

#### Null

`null` represents the **explicit absence of value**. It is a literal value that can be assigned to any variable
whose type includes `null` in a union (e.g. `T|null`). A plain `T` variable **cannot** hold `null`; attempting to
assign `null` to it is a compile-time error.

```nl
string|null name = null;   // OK — type explicitly allows null
string label = null;       // compile-time error — string does not accept null
```

#### Default values

When a fixed-size array is created with `new T[n]`, or when a class property is declared without an explicit
initializer, the following **default values** apply:

| Type category    | Default value |
|------------------|---------------|
| `int`, `byte`    | `0`           |
| `float`          | `0.0`         |
| `bool`           | `false`       |
| `string`         | `""`          |
| Reference types (`T\|null`) | `null` |

A class property of a **non-nullable reference type** (e.g. `private MyObject obj;`) **must** be initialized
either at the declaration site or inside every `construct` path. The compiler rejects any code path where such a
property remains uninitialized after construction.

```nl
class Example {
    private int count;               // default 0
    private string label;            // default ""
    private MyObject|null cache;     // default null
    private MyObject required;       // must be initialized in construct

    public construct(MyObject r) {
        this.required = r;           // OK
    }
}
```

#### Definite assignment (local variables)

Local variables **must** be assigned before their first use. The compiler performs *definite assignment analysis*
(as in Java, C#, and Kotlin) and rejects any code path where a local variable is read before being written.

```nl
int x;
// system.Out.print(x);  // compile-time error — x is not definitely assigned
x = 42;
system.Out.print(x);     // OK
```

This also applies across branches:

```nl
int y;
if (condition) {
    y = 1;
} else {
    y = 2;
}
system.Out.print(y);  // OK — y is assigned on every path
```

```nl
int z;
if (condition) {
    z = 1;
}
// system.Out.print(z);  // compile-time error — z may not be assigned
```

#### Variable shadowing

NL allows **variable shadowing** (as in Java, C#, and Kotlin). An inner declaration hides an outer one of the same name within its scope.

**Local or parameter shadowing a class field.** A local variable or parameter may have the same name as a class property. The unqualified name refers to the local or parameter; the field is accessible via `this.name`.

```nl
class Point {
    private int x;
    private int y;

    public construct(int x, int y) {
        this.x = x;   // parameter x shadows field; this.x accesses the field
        this.y = y;
    }
}
```

**Block-scoped variable shadowing.** A variable declared in an inner block shadows a variable of the same name in an enclosing block. The inner declaration takes precedence within its block.

```nl
void example() {
    int i = 0;
    for (int i = 0; i < 10; i++) {  // loop variable i shadows outer i
        system.Out.print(i);        // refers to loop variable
    }
}
```

Variables are **block-scoped**: a variable declared in a `for` init, an `if` block, or any `{ }` block is visible only within that block.

#### Reserved keyword `undefined`

`undefined` is a **reserved keyword** that cannot be used as an identifier. It does not participate in the type
system and cannot appear in code as a value, type, or comparison operand. The concept it historically represented
(uninitialized state) is handled entirely by the compiler through definite assignment analysis and mandatory
property initialization.

### Union types and explicit nullable

NL supports **union types** in the form `Type1|Type2|…|null`, which can be used for parameter types, return types, and variable declarations. This allows methods to accept or return multiple possible types (including `null`) in an explicit way, which improves refactoring and type safety.

- **Nullable (value or null)**  
  A type that can hold either a value of type `T` or `null` must be written explicitly as **`T|null`**. The suffix **`?` is not accepted**; the language requires the explicit union with `null`.

- **General union types**  
  Unions like **`Type1|Type2|null`** are valid wherever a type is expected (e.g. method parameters, return types, local variables). They are especially useful for parameters and return types when a method can return different types or no value (e.g. optional result, end-of-stream, or multiple result shapes during refactoring).

```nl
string|null line = system.In.readLine();  // null on EOF
string|null ext = system.io.Path.extension(path);  // null if no extension
```

Variables of type `T|null` (or any union containing `null`) can be compared with `null` and used with the nullish coalescing operator `??` or the elvis operator `?:` to provide defaults.

### Type conversions and casting

NL distinguishes **implicit** conversions (no syntax required) from **explicit** conversions (cast required). The cast operator uses C-style syntax: **`(T) expr`** evaluates `expr` and converts the value to type `T` when the conversion is allowed.

#### Implicit conversions

The following conversions are allowed without a cast:

| Source → Target | Description |
|-----------------|-------------|
| `byte` → `int` | Numeric widening. |
| `int` → `float` | Numeric widening. |
| Class → superclass or implemented interface | Upcast; always valid. |
| `T` → `T\|null` | Widening nullability; a non-null value can be assigned to a nullable variable. |

There is **no** implicit conversion between `bool` and numeric types (`int`, `float`, `byte`). This avoids C/C++-style confusion where integers are used as booleans.

#### Explicit conversions (cast required)

| Source → Target | Rule | Runtime behavior |
|-----------------|------|-------------------|
| `float` → `int` | Narrowing; explicit cast required. | Truncation toward zero. Values outside the range of `int` have undefined behavior (or implementation-defined). |
| `int` → `byte` | Narrowing; explicit cast required. | Low-order bits are kept; overflow is implementation-defined if the value does not fit in `byte`. |
| Any → `string` | Allowed if source is a primitive or implements [Stringable](#stringable-interface). | Primitives use built-in string representation; reference types use `toString()`. Otherwise compile-time error. |
| Superclass → subclass | Downcast; allowed at compile-time when the static type could be the target. | **Runtime check**: if the object is not actually an instance of the target type, **`InvalidCastException`** is thrown (see [Exception class hierarchy](#exception-class-hierarchy)). |
| Unrelated class → class | Not allowed. | Compile-time error. |

Casting from `T|null` to `T` is **not** allowed as a simple `(T) expr` when `expr` might be null: the compiler does not allow reducing nullability by cast alone. Use an explicit null check, the nullish coalescing operator `??`, or the elvis operator `?:` instead.

#### Cast to string

The **`(string) expr`** cast follows the same rules as [string concatenation](#string-concatenation): primitives use their built-in string representation; reference types that implement [Stringable](#stringable-interface) use `toString()`; otherwise it is a compile-time error. The result is consistent with `system.Out.print` / `system.Out.println` and with `system.Int.toString`, `system.Float.toString`, and `system.Bool.toString`.

#### Summary table

| Conversion | Implicit | Explicit cast | Notes |
|------------|----------|---------------|-------|
| `byte` → `int` | ✓ | — | |
| `int` → `float` | ✓ | — | |
| `float` → `int` | — | `(int) expr` | Truncation toward zero. |
| `int` → `byte` | — | `(byte) expr` | Low-order bits. |
| Any → `string` | — | `(string) expr` | Primitives or Stringable. |
| Class → superclass | ✓ | — | Upcast. |
| Superclass → subclass | — | `(SubClass) expr` | Runtime check; throws on failure. |
| `T` → `T\|null` | ✓ | — | |
| `bool` ↔ numeric | — | Not supported | No conversion. |

### Auto type

The `auto` keyword enables automatic type deduction, similar to C++. When using `auto`, the compiler automatically
deduces the type of a variable from its initializer expression. This reduces verbosity and makes code more maintainable,
especially when dealing with complex types or template instantiations. The deduced type is determined at compile-time,
ensuring type safety while simplifying the code.

```nl
// Basic type deduction
auto number = 42;              // deduced as int
auto text = "Hello";           // deduced as string
auto value = 3.14;             // deduced as float
auto flag = true;              // deduced as bool

// With function return types
auto result = calculateValue(); // type deduced from calculateValue() return type

// With template types
auto vector = new Vector<int>(1, 2, 3); // deduced as Vector<int>

// With arrays
auto numbers = new int[]{1, 2, 3, 4, 5}; // deduced as int[]

// In loops (const optional; see § Loops for semantics)
for (const auto item : collection) {
    system.Out.print(item);
}
for (auto item : collection) {
    // mutable loop variable
}

// With complex expressions
auto sum = a + b;              // type deduced from the result of a + b
auto product = multiply(x, y); // type deduced from multiply() return type
```

### Typedef

The `typedef` keyword allows you to create type aliases, providing alternative names for existing types. This feature improves code readability and maintainability by giving meaningful names to types, making code more expressive and self-documenting.

Typedefs are scoped to their namespace and can be used anywhere a type is expected. They provide a compile-time alias and do not create new types, meaning they are fully interchangeable with their underlying types.

#### Basic usage

```nl
namespace com.example;

// Simple type aliases
typedef string Name;
typedef int Age;
typedef int[] NumberList;

class Person {
    public Name firstName;
    public Name lastName;
    public Age age;
    public NumberList phoneNumbers;
    
    public construct(Name firstName, Name lastName, Age age) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.age = age;
        this.phoneNumbers = new int[]{};
    }
}

// Typedef and original type are interchangeable
Name name = "John";
string alsoName = name; // OK: Name is just an alias for string

Age personAge = 30;
int sameAge = personAge; // OK: Age is just an alias for int
```

#### With arrays

```nl
namespace com.example;

typedef string[] StringArray;
typedef int[][] Matrix;

class DataProcessor {
    public StringArray process(StringArray input) {
        // Process the array
        return input;
    }
    
    public Matrix createMatrix(int size) {
        return new int[size][size];
    }
}
```

For more advanced use cases with [templates](#template-class) and [anonymous functions](#anonymous-functions), see [Typedef advanced usage](#typedef-advanced-usage).

## Classes

### Basic class

```nl
class Contact
{
    public string lastName;
    public string firstName;
}
```

Usage:

```nl
auto john = new Contact();
john.lastName = "Doe";
john.firstName = "John";
```

### Visibility

NL provides three visibility modifiers to control access to class members: `public`, `protected`, and `private`. These modifiers determine which parts of the code can access a particular member, enabling encapsulation and proper object-oriented design.

- **`public`**: Members marked as `public` are accessible from anywhere in the codebase. They form the public interface of the class and can be accessed by external code, subclasses, and the class itself.

- **`protected`**: Members marked as `protected` are accessible within the class itself and by its subclasses (derived classes). They are not accessible from external code, providing a way to share implementation details with inheritance hierarchies while maintaining encapsulation.

- **`private`**: Members marked as `private` are only accessible within the class itself. They cannot be accessed by subclasses or external code, providing the strongest level of encapsulation. Private members are typically used for internal implementation details that should not be exposed.

```nl
class BaseClass
{
    public string publicField;        // Accessible from anywhere
    protected string protectedField;  // Accessible in this class and subclasses
    private string privateField;      // Accessible only in this class
    
    public void publicMethod() {
        this.publicField = "accessible";
        this.protectedField = "accessible";
        this.privateField = "accessible"; // OK: same class
    }
    
    protected void protectedMethod() {
        this.publicField = "accessible";
        this.protectedField = "accessible";
        this.privateField = "accessible"; // OK: same class
    }
    
    private void privateMethod() {
        this.publicField = "accessible";
        this.protectedField = "accessible";
        this.privateField = "accessible"; // OK: same class
    }
}

class DerivedClass extends BaseClass
{
    public void testAccess() {
        this.publicField = "accessible";     // OK: public member
        this.protectedField = "accessible";  // OK: protected member accessible in subclass
        // this.privateField = "error";       // Error: private member not accessible in subclass
    }
}

// External usage
BaseClass obj = new BaseClass();
obj.publicField = "accessible";        // OK: public member
// obj.protectedField = "error";       // Error: protected member not accessible externally
// obj.privateField = "error";         // Error: private member not accessible externally
```

Visibility modifiers can be applied to properties, methods, constructors (using `construct`), and destructors (using `destruct`). A visibility modifier (`public`, `protected`, or `private`) must always be explicitly specified for each member; omitting the visibility modifier is not permitted.

### Self and type keywords

NL provides three related mechanisms so you don't have to repeat the class name:

- **`this`**: The current instance of the class. Use it to access members and as the return value in method bodies (e.g. `return this` in fluent methods). This is the only way to refer to the instance at the value level.

- **`Self`** (capital S): A type alias for the current class, used when the method **mutates** the current instance and returns it. Use `Self` as the return type for fluent methods and for assignment operators (`+=`, `-=`, etc.) that modify and return the same object. Semantically: "returns this instance, whose type is the current class."

- **`type`** (lowercase): A type alias for the current class, used when the operation **creates** a new object of the same type. Use `type` as the return type for binary operators that return a new instance (e.g. `operator+`), and when instantiating (`new type(...)`). Semantically: "returns or constructs a new instance of the current class."

Keeping both `Self` and `type` makes the intent explicit: `Self` means mutation and return of the same instance; `type` means creation of a new instance. The same keyword `type` is used in templates to declare type parameters; there is no conflict as the contexts are different.

### Super keyword

The `super` keyword provides access to members of the parent class in inheritance hierarchies. It allows derived classes to call methods, access properties, or invoke constructors from their base class, enabling proper code reuse and method overriding patterns.

```nl
class BaseClass {
    protected string value;
    
    public construct(string value) {
        this.value = value;
    }
    
    public void display() {
        system.Out.print("Base: " + this.value);
    }
}

class DerivedClass extends BaseClass {
    public construct(string value) {
        super(value); // Call parent constructor
    }
    
    public void display() {
        super.display(); // Call parent method
        system.Out.print("Derived: " + this.value);
    }
}
```

The `super` keyword can be used to call parent constructors, invoke overridden methods, or access protected members from the base class, ensuring proper initialization and behavior in inheritance relationships.

### Constructors and Destructors

Constructors and destructors use dedicated keywords instead of repeating the class name, making the code more
maintainable and avoiding redundancy.

#### Constructor

```nl
class Contact
{
    public string lastName;
    public string firstName;
    
    public construct() {
        this.lastName = "";
        this.firstName = "";
    }
    
    public construct(string lastName, string firstName) {
        this.lastName = lastName;
        this.firstName = firstName;
    }
}
```

Usage:

```nl
Contact john = new Contact("Doe", "John");
Contact empty = new Contact();
```

#### Destructor

```nl
class Resource
{
    private system.io.FileHandle handle;
    
    public construct() {
        this.handle = system.io.File.open("data.txt");
    }
    
    public destruct() {
        this.handle.close();
    }
}
```

The destructor is automatically called when the object becomes unreachable and is reclaimed by the garbage collector. The exact timing is implementation-defined (see [vm.md § Garbage collection contract](vm.md#garbage-collection-contract)).

### Class methods

#### Basic methods

```nl
class MyClass
{
    public void myMethod(int arg1, string arg2)
    {
        // block
    }
}
```

#### Static methods

```nl
class MyClass
{
    public static void aStaticMethod() 
    {
        // Callable with MyClass.aStaticMethod().
        // Can only access static members.
    }
}
```

#### Const methods and parameters

The `const` keyword provides immutability guarantees: when applied to a method, it prevents the method from modifying
the object's state; when applied to a parameter, it prevents the parameter from being modified within the method body;
when applied to a local variable, it prevents the variable from being reassigned after its initial assignment.
This helps ensure data integrity and enables safe sharing of objects without unintended side effects.

```nl
class MyClass
{
    public int aConstMethod() const
    {
        // Cannot modify the properties of this object.
        return this.aMemberValue;
    }

    public void withConstParam(int arg1, const MyOtherObject arg2)
    {
        // Can only use const methods of MyOtherObject.
    }

    public void withConstNative(int arg1, const int arg2) 
    {
        arg1 = 2; // OK
        arg2 = 2; // Error: cannot modify a const value.
    }

    public void example()
    {
        const int x = 42;
        x = 10;  // Error: cannot modify a const variable.
    }
}
```

#### Parameter passing semantics

Parameters are passed by **value** or by **reference-value** by default, depending on the type:

- **Scalar types** (`int`, `float`, `bool`, `byte`, etc.): passed **by value**. The method receives a copy; assignments to the parameter do not affect the caller's variable.
- **Object types** (class instances): passed **by reference-value**. The method receives a copy of the reference; it can mutate the object's state, but reassigning the parameter (e.g. `param = new MyClass()`) does not change the caller's variable.

To allow a method to modify the **caller's variable** (e.g. for a `swap`), use the **`ref`** modifier. The parameter becomes a true reference to the caller's variable. The caller must also use `ref` at the call site, making the intent explicit.

| Declaration    | Meaning |
|----------------|--------|
| `T param`      | By value (scalar) or by reference-value (object); modifiable in the method. |
| `const T param`| Same, but the parameter cannot be modified in the method body. |
| `ref T param`  | By reference: the method can read and write the caller's variable. |
| `const ref T param` | By reference, read-only: the method can read the caller's variable but not modify it (avoids copying large values). |

```nl
class Utils {
    template <type T>
    public static void swap(ref T a, ref T b) {
        T temp = a;
        a = b;
        b = temp;
    }
}

int x = 10;
int y = 20;
Utils.swap(ref x, ref y); // x = 20, y = 10
```

Rules for `ref` parameters:

- At the call site, arguments must be variables (or writable expressions); the `ref` keyword is required.
- Optional parameters cannot be declared with `ref` (the caller must supply a variable).
- `ref` is independent of `const`: use `const ref` for read-only reference parameters.

### Named parameters

NL supports named parameters, allowing you to specify method arguments by name rather than by position. This feature improves code readability, especially when methods have many parameters or when you want to make the intent of each argument clear. Named parameters work seamlessly with optional parameters (parameters with default values) and can be used with methods, constructors, and anonymous functions.

#### Optional parameters

Parameters can be given default values, making them optional. Optional parameters must be placed after all required parameters in the method signature.

```nl
class Thing {
    // This method has many parameters, you can use named arguments to call it
    public static void doSomething(
        int a, // required parameter   
        string b, // required parameter
        float c = 0.0, // optional parameter
        bool d = false // optional parameter
    ) {
        // Doing something with a, b, c, d
    }
}
```

#### Using named parameters

When calling a method, you can use named parameters by specifying the parameter name followed by a colon and the value. This allows you to:

- Skip optional parameters you don't need
- Pass arguments in any order (as long as required parameters are provided)
- Improve code readability by making the purpose of each argument explicit
- Mix positional and named arguments

```nl
// Using positional arguments (traditional way)
Thing.doSomething(1, "Hello", 3.14, true);

// Using named parameters - all arguments
Thing.doSomething(
    a: 1,
    b: "Hello",
    c: 3.14,
    d: true
);

// Using named parameters - skipping optional parameters
Thing.doSomething(
    a: 1,
    b: "Hello"
    // c and d use their default values (0.0 and false)
);

// Using named parameters - mixing order
Thing.doSomething(
    b: "Hello",
    a: 1,
    d: true
    // c uses its default value (0.0)
);

// Mixing positional and named arguments
// Positional arguments must come first, followed by named arguments
Thing.doSomething(1, "Hello", c: 3.14); // a=1, b="Hello", c=3.14, d=false
Thing.doSomething(1, b: "Hello", d: true); // a=1, b="Hello", c=0.0, d=true
```

#### Named parameters with constructors

Named parameters can also be used when calling constructors, making object initialization more readable:

```nl
class Contact {
    public string lastName;
    public string firstName;
    public int age;
    
    public construct(string lastName, string firstName, int age = 0) {
        this.lastName = lastName;
        this.firstName = firstName;
        this.age = age;
    }
}

// Usage with named parameters
auto john = new Contact(lastName: "Doe", firstName: "John");
auto jane = new Contact(firstName: "Jane", lastName: "Smith", age: 30);

// Mixing positional and named arguments
auto bob = new Contact("Smith", firstName: "Bob", age: 25);
```

#### Named parameters with anonymous functions

Named parameters can be used when calling anonymous functions, providing the same flexibility as with regular methods:

```nl
// Define an anonymous function with optional parameters
auto process = (int value, int multiplier = 1, int offset = 0) => int {
    return (value * multiplier) + offset;
};

// Using named parameters
int result1 = process(value: 10, multiplier: 2, offset: 5); // 25
int result2 = process(10, offset: 5); // 15 (multiplier uses default 1)
int result3 = process(offset: 5, value: 10); // 15 (order doesn't matter)

// Passing anonymous functions with named parameters
void applyOperation(int[] numbers, (int, int) => int operation) {
    for (int i = 0; i < numbers.length(); i++) {
        numbers[i] = operation(numbers[i], multiplier: 2);
    }
}

auto multiply = (int value, int multiplier = 1) => int {
    return value * multiplier;
};

applyOperation(numbers, multiply); // multiply will be called with multiplier: 2
```

#### Rules and restrictions

- Required parameters (without default values) must be provided either positionally or by name
- Optional parameters can be omitted and will use their default values
- When mixing positional and named arguments, all positional arguments must come before any named arguments
- Named parameters can be specified in any order among themselves
- Default values must be compile-time constants
- All required parameters must be provided, either positionally or by name

```nl
class Calculator {
    public int compute(
        int base,
        int multiplier = 1,
        int offset = 0
    ) {
        return (base * multiplier) + offset;
    }
}

// All valid calls:
Calculator calc = new Calculator();

calc.compute(10);                              // base=10, multiplier=1, offset=0
calc.compute(10, 2);                           // base=10, multiplier=2, offset=0
calc.compute(10, 2, 5);                        // base=10, multiplier=2, offset=5
calc.compute(base: 10);                        // base=10, multiplier=1, offset=0
calc.compute(base: 10, offset: 5);             // base=10, multiplier=1, offset=5
calc.compute(10, offset: 5);                   // base=10, multiplier=1, offset=5
calc.compute(10, multiplier: 2, offset: 5);   // base=10, multiplier=2, offset=5
calc.compute(offset: 5, base: 10, multiplier: 2); // base=10, multiplier=2, offset=5 (order doesn't matter)
```

Named parameters make method calls more self-documenting and reduce errors when working with methods that have many parameters, especially when most of them are optional.

### Fluent methods

```nl
class Book {
    private string title;
    private string author;
    private int publishedYear;
    
    public Self setTitle(string title) {
        this.title = title;
        return this;
    }
    public Self setAuthor(string author) {
        this.author = author;
        return this;
    }
    public Self setPublishedYear(int year) {
        this.publishedYear = year;
        return this;
    }
    public Self save() {
        // save in database, or whatever.
        return this;
    }
}
```

Usage:

```nl
{
    auto book = new Book()
        .setTitle("1984")
        .setAuthor("George Orwell")
        .setPublishedYear(1950)
        .save();
}
```

### Readonly

The `readonly` keyword enforces immutability at different levels: when applied to a class, all properties of instances
become immutable after construction; when applied to a property, only that specific property cannot be modified after
the object is constructed. Readonly properties can only be assigned during object initialization (in the constructor),
ensuring that their values remain constant throughout the object's lifetime.

#### Class

```nl
class readonly ValueObject {
    public int value;
    public construct(int value) {
        this.value = value; // Readonly properties can only be assigned in the constructor
    }
}

// usage
ValueObject obj = new ValueObject(1);
system.Out.print(obj.value); // OK: 1
obj.value = 12; // Error: Cannot modify a property of a readonly class.
```

#### Property

```nl
class ValueObject {
    public readonly int value;
    public int value2;
    public construct(int value, int value2) {
        this.value = value; // Readonly properties can only be assigned in the constructor
        this.value2 = value2;
    }
}

// usage
ValueObject obj = new ValueObject(1, 2);
system.Out.print(obj.value); // OK: 1
obj.value = 12; // Error: Cannot modify a readonly property.
system.Out.print(obj.value2); // OK: 2
obj.value2 = 12; // OK: Property is not readonly and is public.
```

### Nodiscard

The `nodiscard` keyword indicates that the return value of a function should not be ignored. When a function marked with `nodiscard` is called and its return value is not used, the compiler emits a warning. This helps prevent common programming errors where important return values (such as error codes, resource handles, or computed results) are accidentally discarded.

The `nodiscard` modifier can be applied to methods and functions to enforce that callers must handle or use the returned value. This is particularly useful for functions that return error codes, status indicators, or other values that represent important state information.

```nl
class Result
{
    private bool success;
    private string message;
    
    public construct(bool success, string message) {
        this.success = success;
        this.message = message;
    }
    
    public bool isSuccess() {
        return this.success;
    }
    
    public string getMessage() {
        return this.message;
    }
}

class FileHandler
{
    public nodiscard Result openFile(string filename) {
        // Attempt to open file
        if (system.io.File.exists(filename)) {
            return new Result(true, "File opened successfully");
        }
        return new Result(false, "File not found");
    }
    
    public nodiscard int computeValue(int input) {
        // Important computation that should not be ignored
        return input * 2 + 10;
    }
}

// Usage
FileHandler handler = new FileHandler();

// OK: return value is used
auto result = handler.openFile("data.txt");
if (result.isSuccess()) {
    system.Out.print("Success");
}

// Warning: return value is ignored
handler.openFile("data.txt"); // Warning: nodiscard return value is discarded

// OK: return value is used
int value = handler.computeValue(5);
system.Out.print(value);

// Warning: return value is ignored
handler.computeValue(5); // Warning: nodiscard return value is discarded
```

The `nodiscard` modifier helps catch potential bugs at compile-time by ensuring that important return values are not accidentally ignored, improving code safety and maintainability.

### Template class

```nl
template <type T>
class Vector 
{
    private T x;
    private T y;
    private T z;

    public construct(T x, T y, T z)
    {
        this.x = x;
        this.y = y;
        this.z = z;
    }

    public T getX() 
    {
        return this.x;
    }

    public Self setX(T value)
    {
        this.x = value;
        return this;
    }
    
    public type operator+(const type vec)
    {
        return new type(this.x + vec.x, this.y + vec.y, this.z + vec.z);
    }
}
```

Usage:

```nl
Vector<int> v1 = new Vector<int>(1, 2, 3);
Vector<float> v2 = new Vector<float>(.1, .2, .3);
```

#### Bounded type parameters

Type parameters can be constrained with `extends` to require that the concrete type is a subtype of a given class or interface. This enables earlier, clearer compile-time errors and documents the template contract.

Syntax: `template <type T extends Bound>` where `Bound` is a class or interface. The concrete type argument must implement the interface (or extend the class) at instantiation time.

```nl
// Stringable is defined in § Extends, Implements — requires string toString()
template <type T extends Stringable>
class Formatter {
    public string format(const T value) {
        return value.toString();
    }
}

class Person implements Stringable {
    public string name;
    public string toString() { return this.name; }
}

Formatter<Person> f = new Formatter<Person>();  // OK: Person implements Stringable
// Formatter<int> g = new Formatter<int>();    // Error: int does not implement Stringable
```

- **Reference types:** The concrete type must implement the interface or extend the bound class.
- **Primitive types:** Primitives (`int`, `float`, `bool`, `byte`) do not implement interfaces; they satisfy only unconstrained type parameters or bounds that explicitly include them (e.g. via built-in rules, if any).
- **Multiple bounds:** Not supported; use a single class or interface as the bound.

Bounded type parameters apply to both template classes and template methods. See [compiler.md § Template instantiation](compiler.md#template-instantiation) for verification rules.

### Template methods

In addition to template classes, NL supports template methods, which allow you to define methods that work with generic types without making the entire class a template. Template methods enable you to write type-safe, reusable code for operations that should work with multiple types.

Template methods are declared using the `template <type T>` or `template <type T extends Bound>` syntax before the method signature. The type parameter can then be used in the method's parameters, return type, and body. [Bounded type parameters](#bounded-type-parameters) apply to template methods as well.

```nl
class Utils {
    // Template method to swap two values (uses ref so the caller's variables are updated)
    template <type T>
    public static void swap(ref T a, ref T b) {
        T temp = a;
        a = b;
        b = temp;
    }
    
    // Template method to find the maximum of two values
    template <type T>
    public static T max(const T a, const T b) {
        return a > b ? a : b;
    }
    
    // Template method that works with different numeric types
    template <type T>
    public static T clamp(const T value, const T min, const T max) {
        if (value < min) return min;
        if (value > max) return max;
        return value;
    }
}

// Usage:
int x = 10;
int y = 20;
Utils.swap(ref x, ref y); // x = 20, y = 10

float a = 3.14;
float b = 2.71;
float maximum = Utils.max(a, b); // maximum = 3.14

int value = 150;
int clamped = Utils.clamp(value, 0, 100); // clamped = 100
```

Template methods are particularly useful when you need generic functionality within a non-template class, or when only specific methods of a class need to be generic while the rest of the class works with concrete types.

### Extends, Implements

```nl
class Foo {
    protected void doSomethingProtected() {
    }
}
interface Stringable {
    public string toString();
}
class Bar extends Foo implements Stringable 
{
    public string toString()
    {
        this.doSomethingProtected();
        return "Bar";
    }
}

```

#### Stringable interface

The **Stringable** interface is the standard contract for converting reference types to string. A class that implements Stringable must provide a method `string toString()`.

When a value is converted to string (in [string concatenation](#string-concatenation) or via the [cast to string](#other-operators) `(string) expr`), the runtime uses this interface for reference types: if the static type of the value implements Stringable, the result is `expr.toString()`. Primitives (`int`, `float`, `bool`) use built-in string representation and do not implement interfaces.

```nl
interface Stringable {
    public string toString();
}
```

### Abstract classes and methods

An **abstract class** cannot be instantiated directly. It may declare **abstract methods** — methods without a body that must be implemented by concrete subclasses. A class that declares or inherits any abstract method must itself be declared `abstract`.

```nl
abstract class Shape {
    public abstract float area();
    public abstract float perimeter();
}

class Rectangle extends Shape {
    private float width;
    private float height;

    public construct(float width, float height) {
        this.width = width;
        this.height = height;
    }

    public float area() {
        return this.width * this.height;
    }

    public float perimeter() {
        return 2 * (this.width + this.height);
    }
}
```

Rules:
- An abstract class cannot be instantiated with `new`.
- An abstract method has no body (no `{ }` block); it ends with `;`.
- A concrete class that extends an abstract class must implement all inherited abstract methods.
- Interface methods are implicitly abstract (no body in the interface).
- An abstract class may have constructors; they are invoked when a concrete subclass is instantiated via `super(...)`.

### Final classes and methods

The `final` modifier restricts inheritance and overriding:

- **`final class`** — The class cannot be extended. No other class may use `extends` with it.
- **`final method`** — The method cannot be overridden in subclasses.

```nl
final class StringUtils {
    public static string trim(string s) { /* ... */ }
}
// class Extended extends StringUtils { }  // Error: cannot extend final class

class Base {
    public final void cannotOverride() { /* ... */ }
}
class Derived extends Base {
    // public void cannotOverride() { }  // Error: cannot override final method
}
```

Rules:
- A `final` class cannot have abstract methods.
- `final` and `abstract` are mutually exclusive on a method (a method cannot be both).
- Private methods are implicitly final (they are not visible to subclasses and thus cannot be overridden).

### Virtual method dispatch

All non-static, non-private instance methods participate in **virtual dispatch** by default (equivalent to Java's behavior). When a method is called on an object, the implementation is chosen at runtime based on the object's actual class, not the static type of the variable.

No explicit `virtual` keyword is required. Methods are virtual by default. The `final` modifier opts out of overriding (and thus of further virtual dispatch in subclasses).

### Cloneable interface

Object cloning is provided via the **Cloneable** interface. A class that implements Cloneable must provide a method `Self clone()` returning a copy of the object. The default semantics are **shallow copy**: reference-type fields are copied by reference, not recursively cloned. For deep copy semantics, implement a custom method (e.g. `deepClone()`).

```nl
interface Cloneable {
    public Self clone();
}

class Point implements Cloneable {
    public int x;
    public int y;

    public construct(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public Self clone() {
        return new Point(this.x, this.y);
    }
}
```

Usage: `auto copy = original.clone();` — no dedicated `clone` keyword. The `clone()` method is an ordinary instance method.

### ValueEquatable interface

Structural (value-based) equality of objects is provided via the **ValueEquatable** interface. A class that implements ValueEquatable must provide:

- **`bool valueEquals(const Self|null other)`** — returns `true` if `this` and `other` are structurally equal. Returns `false` if `other` is `null` or if the runtime type of `other` differs from `this`. The implementation defines what "structurally equal" means (typically: same field values).
- **`int valueHash()`** — returns a hash code consistent with `valueEquals`: if `a.valueEquals(b)` then `a.valueHash() == b.valueHash()`. The reverse is not required (hash collisions are allowed).

The built-in `==` operator on references compares **identity** (same object instance), not value. Use `valueEquals()` when you need structural equality — for example, to use objects as `system.Map` keys with value-based lookup.

```nl
interface ValueEquatable {
    public bool valueEquals(const Self|null other);
    public int valueHash();
}

class Point implements ValueEquatable {
    public int x;
    public int y;

    public construct(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public bool valueEquals(const Self|null other) {
        if (other == null) return false;
        return this.x == other.x && this.y == other.y;
    }

    public int valueHash() {
        return 31 * this.x + this.y;
    }
}

// Usage:
auto p1 = new Point(1, 2);
auto p2 = new Point(1, 2);
p1 == p2;           // false — different instances (identity)
p1.valueEquals(p2); // true  — same values (structural)
```

Primitive types (`int`, `float`, `bool`, `byte`) and `string` use value equality by default; they do not implement ValueEquatable. `system.Map<K, V>` uses `valueEquals` and `valueHash` for key lookup when `K` is a reference type that implements ValueEquatable; otherwise it uses reference identity (see [stdlib.md § system.Map](stdlib.md#systemmap)).

## Enums

### Basic enums

```nl
enum States
{
    Waiting,
    Loading,
    Started,
    Finished, // final trailing comma is valid
}
```

### Typed enums (string or int)

```nl
enum Status: int
{
    OK = 200,
    NotFound = 400,
    InternalServerError = 500,
}
```

Usages

```nl
Status status = Status.OK;
system.Out.print(status.value); // 200
```

### Enums methods

```nl
static Self from(string value)
static Self|null tryFrom(string value)
```

- **`from(value)`** : converts the string to the corresponding enum value. Throws **`IllegalArgumentException`** (subclass of `RuntimeException`) if `value` does not match any enum case.
- **`tryFrom(value)`** : same conversion, but returns **`null`** instead of throwing an exception if `value` does not match any case.

Example:

```nl
Status status = Status.from("NotFound");
if (status != Status.NotFound) { /* expected NotFound */ }

auto maybe = Status.tryFrom("NotFound");
if (maybe == null || maybe != Status.NotFound) { /* expected non-null NotFound */ }

auto invalid = Status.tryFrom("Invalid");
if (invalid != null) { /* expected null */ }

// Status.from("Invalid"); // throws IllegalArgumentException
```

## Control Structures

### Conditionals

```nl
if (condition) {
    // block
} else {
    // alternative block
}
```

### Loops

`break` exits the loop immediately; `continue` skips the rest of the current iteration and proceeds to the next one.

```nl
while (condition) {
    // block
    break;
}

for (init; condition; increment) {
    // block
    // break; // available
}

for (const auto item : collection) {
    // item type is deduced from collection element type
}

for (auto item : collection) {
    // item is mutable; use when the loop body needs to modify the loop variable
}
```

**For-each loop — copy semantics.** The loop variable holds a **copy** of each element: for value types (`int`, `float`, `bool`, `byte`), it is a copy of the value; for reference types (objects, `string`), it is a copy of the reference. Reassigning the loop variable never affects the collection. For reference types, calling mutating methods on the loop variable modifies the referred-to object; `const` on the loop variable prevents such calls.

**For-each loop — `const` optional.** Both forms are valid:

- `for (const auto item : collection)` — the loop variable is read-only; assignments to `item` and calls to non-const methods on `item` are rejected.
- `for (auto item : collection)` — the loop variable is mutable; the loop body may modify it (when the element type permits).

**Implicit const in const context.** When iterating over a collection that is read-only in the current scope, the loop variable is implicitly non-modifiable (as if `const auto` had been written). This applies when:

1. The loop appears inside a **const method** and the collection is a property of `this` (e.g. `for (auto item : this.items)`).
2. The collection is a **const parameter** or **const ref parameter** (e.g. `void f(const int[] arr) { for (auto x : arr) { ... } }`).

In these cases, the compiler treats the loop variable as read-only and rejects any modification. This ensures deep immutability: a const method cannot mutate the object's logical state, including elements of its collections.

### Switch/Match

The `switch` statement uses **fall-through** semantics: without a `break` statement, execution continues into the next `case` body. Use `break` to exit the switch block after each case.

```nl
switch (value) {
    case 1:
        doSomething();
        break;
    case 2:
        doSomethingElse();
        break;
    default:
        doNothing();
        break;
}
```

```nl
string result = match(value) {
    200: "ok",
    404: "not found",
    default: Config.getDefaultValue(),
};
```

## Anonymous Functions

NL supports anonymous functions (also called lambdas or closures) that allow you to define inline functions without explicitly naming them. Anonymous functions are first-class citizens and can be assigned to variables, passed as arguments, or returned from other functions.

### Basic syntax

Anonymous functions use the `=>` operator (consistent with `match` expressions) to separate parameters from the function body. The return type can be explicitly specified or automatically deduced by the compiler.

```nl
// Simple anonymous function with type deduction
auto add = (int a, int b) => { return a + b; };
int result = add(5, 3); // result = 8

// Single expression (no braces needed)
auto multiply = (int a, int b) => a * b;

// Explicit return type
auto divide = (int a, int b) => float { return (float)a / b; };

// No parameters
auto getValue = () => { return 42; };
```

### Variable capture

Anonymous functions can capture variables from their enclosing scope. Variables are automatically captured by reference, allowing the function to access and modify variables from the surrounding context.

```nl
int multiplier = 10;

// Variables are captured by reference from the enclosing scope
auto multiplyBy = (int value) => { return value * multiplier; };
int result = multiplyBy(5); // result = 50

// Multiple variables can be captured
int x = 5;
int y = 3;
auto addXY = (int z) => { return x + y + z; };
int result2 = addXY(2); // result2 = 10

// Captured variables can be modified
int counter = 0;
auto increment = () => { counter++; };
increment();
system.Out.print(counter); // prints 1
```

### Const parameters

Parameters can be marked as `const` to prevent modification within the function body, consistent with regular method parameters.

```nl
auto process = (const string text) => string {
    // text cannot be modified here
    return text.toUpperCase();
};

auto sum = (const int[] numbers) => int {
    int total = 0;
    for (const auto n : numbers) {
        total += n;
    }
    return total;
};
```

### Exception handling

Anonymous functions can declare exceptions using the `throws` keyword, following the same static exception checking rules as regular methods.

```nl
auto riskyOperation = (string|null input) => throws Exception {
    if (input == null) {
        throw new Exception("Input is null");
    }
    return input.length();
};

// Usage requires exception handling
try {
    int length = riskyOperation("test");
}
catch (Exception ex) {
    system.Out.print(ex.message);
}
```

### Function type assignment

Anonymous functions can be assigned to variables with explicit function types. Function types follow the syntax `(param_types) => return_type` and can include exception declarations.

```nl
// Explicit function type
(int, int) => int addFunction = (int a, int b) => a + b;

// Function type with exceptions
(string|null) => int throws Exception lengthFunction = (string|null s) => throws Exception {
    if (s == null) {
        throw new Exception("String is null");
    }
    return s.length();
};

// Function type as parameter
void processNumbers(int[] numbers, (int) => int transformer) {
    for (int i = 0; i < numbers.length(); i++) {
        numbers[i] = transformer(numbers[i]);
    }
}

// Usage
auto square = (int x) => x * x;
processNumbers(numbers, square);

// Function type as return value
(int) => int createMultiplier(int factor) {
    return (int x) => x * factor;
}

auto multiplyBy5 = createMultiplier(5);
int result = multiplyBy5(10); // result = 50
```

### Common use cases

Anonymous functions are particularly useful with collection operations and callback patterns:

```nl
int[] numbers = new int[]{1, 2, 3, 4, 5};

// Map operation
auto doubled = numbers.map((int n) => n * 2);

// Filter operation
auto evens = numbers.filter((int n) => n % 2 == 0);

// ForEach operation
numbers.forEach((int n) => { system.Out.print(n); });

// Sort with custom comparator
numbers.sort((int a, int b) => a - b);

// Find operation
auto found = numbers.find((int n) => n > 3);
```

### Type deduction

The `auto` keyword can be used to let the compiler deduce the function type, making code more concise while maintaining type safety:

```nl
// Type is deduced as (int, int) => int
auto add = (int a, int b) => a + b;

// Type is deduced as (string) => string
auto greet = (string name) => "Hello, " + name;
```

Anonymous functions provide a powerful and expressive way to write concise, functional-style code while maintaining NL's emphasis on type safety and compile-time checking.

### Typedef advanced usage

The `typedef` keyword becomes particularly powerful when combined with [templates](#template-class) and [anonymous functions](#anonymous-functions). This section covers advanced use cases that leverage these features.

#### Typedef with templates

Typedefs are especially useful for simplifying template instantiations, making code more readable and easier to maintain:

```nl
namespace com.example.math;

template <type T>
class Vector {
    private T x;
    private T y;
    private T z;
    
    public construct(T x, T y, T z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }
}

// Typedef for simplifying template usage
typedef Vector<int> IntVector;
typedef Vector<float> FloatVector;

class Example {
    public IntVector position;
    public FloatVector velocity;
    
    public construct() {
        this.position = new IntVector(0, 0, 0);
        this.velocity = new FloatVector(0.0, 0.0, 0.0);
    }
}
```

#### Typedef with function types

Typedefs can also be used to create aliases for function types, making function signatures more readable and reusable:

```nl
namespace com.example.project;

// Typedef for a function type
typedef (int, int) => int BinaryOperation;

class Calculator {
    private int value1;
    private int value2;
    
    public construct(int value1, int value2) {
        this.value1 = value1;
        this.value2 = value2;
    }
    
    public int compute(BinaryOperation operation) {
        return operation(this.value1, this.value2);
    }
    
    public void applyOperation(BinaryOperation operation, float factor) {
        int result = operation(this.value1, (int)(this.value2 * factor));
        system.Out.print(result);
    }
}

// Usage
auto calc = new Calculator(5, 3);

// Using with simple anonymous function
int product = calc.compute((int a, int b) => a * b);

// Using with anonymous function block
int complex = calc.compute((int a, int b) => {
    return (a * a) * (b / 2);
});

// Using the typedef directly
BinaryOperation multiply = (int a, int b) => a * b;
int result = calc.compute(multiply);
```

#### Typedef with function types and exceptions

Typedefs can also include exception declarations for function types:

```nl
namespace com.example.io;

typedef (string) => int throws Exception StringProcessor;

class FileHandler {
    public void processFile(string filename, StringProcessor processor) throws Exception {
        string content = system.io.File.readAllText(filename);
        int result = processor(content);
        system.Out.print(result);
    }
}

// Usage
auto handler = new FileHandler();

StringProcessor lengthProcessor = (string s) => throws Exception {
    if (s == null) {
        throw new Exception("String is null");
    }
    return s.length();
};

handler.processFile("data.txt", lengthProcessor);
```

Typedefs help make code more expressive and maintainable by providing semantic meaning to complex type expressions while maintaining full type safety.

## Operators

NL provides a comprehensive set of operators for arithmetic, comparison, logical operations, and more.

See also [operator overloading section](#operator-overloading).

### Arithmetic operators

- `+` (addition), `-` (subtraction), `*` (multiplication), `/` (division), `%` (modulo)
- `++` (increment), `--` (decrement) - prefix and postfix variants

### String concatenation

The `+` operator performs string concatenation when at least one operand is of type `string`. The other operand is converted to its **string representation** before concatenation.

- `string + string` → concatenation of the two strings.
- `string + int`, `string + float`, `string + bool` → the numeric or boolean value is converted to its string representation (same as used by `system.Out.print` / `system.Out.println` and by `system.Int.toString`, `system.Float.toString`, `system.Bool.toString`), then concatenated.
- `int + string`, `float + string`, `bool + string` → same conversion of the first operand, then concatenation.
- If one operand is a reference type that implements the [Stringable](#stringable-interface) interface, its `toString()` method is called to obtain the string representation; the result is then concatenated with the other operand.

The string representation of `int`, `float`, and `bool` is implementation-defined but must be consistent across concatenation, the `(string)` cast, and the overloads of `system.Out.print` / `system.Out.println`.

### Comparison operators

- `==` (equality), `!=` (inequality) — For primitives and `string`: value equality. For references: **identity** (same object instance). For value-based equality of objects, use the [ValueEquatable](#valueequatable-interface) interface and `valueEquals()`.
- `<` (less than), `>` (greater than)
- `<=` (less than or equal), `>=` (greater than or equal)
- `<=>` (spaceship operator) - returns `-1` if left < right, `0` if equal, `1` if left > right

### Logical operators

- `&&` (logical AND), `||` (logical OR), `!` (logical NOT)

### Assignment operators

- `=` (assignment)
- `+=`, `-=`, `*=`, `/=`, `%=` (compound assignment)

### Other operators

- `[]` (array/collection access)
- `.` (member access)
- `new` (object instantiation)
- **`(T) expr`** (cast) — explicit type conversion. See [Type conversions and casting](#type-conversions-and-casting) for allowed conversions, implicit vs explicit rules, and runtime behavior. The cast to string **`(string) expr`** converts a value to string using the same rules as [string concatenation](#string-concatenation): primitives use their built-in string representation; reference types that implement [Stringable](#stringable-interface) use `toString()`; otherwise it is a compile-time error.

### Conditional operators

NL provides three conditional operators that allow concise conditional expressions and null/falsy value handling.

#### Ternary operator

The ternary operator `? :` provides a compact way to express conditional values. It evaluates a condition and returns one of two values based on whether the condition is true or false.

```nl
bool a = false;
system.Out.print(a ? "true" : "false"); // prints "false"

int x = 10;
int y = x > 5 ? 100 : 200; // y = 100

string message = isLoggedIn ? "Welcome back" : "Please log in";
```

The ternary operator has the form: `condition ? value_if_true : value_if_false`

- The condition is evaluated first
- If the condition is `true`, the expression evaluates to `value_if_true`
- If the condition is `false`, the expression evaluates to `value_if_false`
- Both branches must have compatible types (the compiler ensures type safety)

#### Nullish coalescing operator

The nullish coalescing operator `??` provides a default value when the left operand is `null`. Unlike the elvis operator, it only checks for `null`, not other falsy values.

```nl
string|null a = null;
system.Out.print(a ?? "is null"); // prints "is null"

string|null b = getString();
system.Out.print(b ?? "default"); // prints b if b is not null, otherwise "default"

string name = user.getName() ?? "Anonymous";
int count = getCount() ?? 0;
```

The nullish coalescing operator has the form: `value ?? default_value`

- If `value` is `null`, the expression evaluates to `default_value`
- If `value` is not `null`, the expression evaluates to `value`
- Only `null` triggers the default value; `false` or `0` do not
- Useful for providing fallback values when dealing with nullable types

#### Elvis operator

The elvis operator `?:` provides a default value when the left operand is falsy. A value is considered falsy if it is `false`, `null`, or `0` (for numeric types).

```nl
int a = 0;
system.Out.print(a ?: "not applicable"); // prints "not applicable" (0 is falsy)

bool flag = false;
string result = flag ?: "not applicable"; // result = "not applicable"

string|null obj = null;
string name = obj ?: "default"; // name = "default"

int value = 42;
system.Out.print(value ?: "not applicable"); // prints 42 (value is truthy)
```

The elvis operator has the form: `value ?: default_value`

- If `value` is falsy (`false`, `null`, or `0`), the expression evaluates to `default_value`
- If `value` is truthy (any other value), the expression evaluates to `value`
- More permissive than nullish coalescing as it handles multiple falsy conditions
- Useful for providing defaults when dealing with values that might be zero, false, or null

#### Operator precedence and usage

Conditional operators have different precedence levels:

- Ternary operator `? :` has lower precedence than most operators
- Nullish coalescing `??` has lower precedence than ternary but higher than assignment
- Elvis operator `?:` has the same precedence as nullish coalescing

When mixing operators, use parentheses to clarify intent:

```nl
// Ternary with nullish coalescing
string result = (condition ? getValue() : null) ?? "default";

// Elvis with ternary
int value = (x > 0 ? x : 0) ?: -1;

// Nested conditionals
string message = user != null 
    ? (user.isActive() ? "Active user" : "Inactive user")
    : "No user";
```

## Operator Overloading

Classes can overload operators to provide custom behavior for built-in operators such as `+`, `-`, `*`, `/`, `==`, etc.
Operator overloading allows objects to be used with natural syntax while maintaining type safety. Binary operators
like `+` typically create and return a new object without modifying the operands, while compound assignment operators
like `+=` modify the left-hand operand and return a reference to it. Operators can be overloaded with different
parameter types, enabling operations like adding a vector to another vector or adding a scalar to a vector.

The `type` keyword is used for binary operators that return a new object, while `Self` is used for assignment operators
that modify and return the current instance.

```nl
class Vector2 {
    public int x;
    public int y;
    
    public construct() {
        this.x = 0;
        this.y = 0;
    }
    
    public construct(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    // Binary operator: returns a new type
    public type operator+(const Vector2 vec) {
        return new type(this.x + vec.x, this.y + vec.y);
    }
    
    // Binary operator with different parameter type
    public type operator+(const int scalar) {
        return new type(this.x + scalar, this.y + scalar);
    }
    
    // Assignment operator: modifies and returns Self
    public Self operator+=(const Vector2 vec) {
        this.x += vec.x;
        this.y += vec.y;
        return this;
    }
}

// Usage:

auto p1 = new Vector2(1, 2);
auto p2 = new Vector2(3, 4);
auto p3 = p1 + p2 + 1;
system.Out.print(p3.x); // 5 (1 + 3 + 1)
system.Out.print(p3.y); // 7 (2 + 4 + 1)
p3 += new Vector2(2, 3);
system.Out.print(p3.x); // 7 (5 + 2)
system.Out.print(p3.y); // 10 (7 + 3)
```

## Exceptions

NL enforces static exception checking with a two-tier exception system. Checked exceptions must be declared in method signatures using the `throws` keyword, while runtime exceptions do not require declaration. This ensures that recoverable errors are explicitly handled while keeping common operations concise.

### Exception class hierarchy

Pseudo code of the Exception class hierarchy

```nl
class readonly Exception {
    public construct(string what);
    public string message;
    public ExecutionPoint[] stackTrace;
}

class readonly ExecutionPoint {
    public int line;
    public string file;
}

class readonly RuntimeException extends Exception {
    // Runtime exceptions represent programming errors and do not require throws declaration
}

class readonly ArithmeticException extends RuntimeException {
    // Division by zero, overflow, underflow
}

class readonly IndexOutOfBoundsException extends RuntimeException {
    // Array or collection index out of bounds
}

class readonly NullPointerException extends RuntimeException {
    // Null reference access
}

class readonly InvalidCastException extends RuntimeException {
    // Invalid cast at runtime (e.g. downcast to wrong subclass)
}

// Runtime exceptions used by the standard library (parsing, invalid arguments)
class readonly NumberFormatException extends RuntimeException {
    // Invalid numeric string format (e.g. system.Int.parseInt, system.Float.parseFloat)
}
class readonly IllegalArgumentException extends RuntimeException {
    // Invalid argument (e.g. enum.from with unknown value, system.time.TimeZone.get with unknown ID)
}

// Checked exceptions used by the standard library (I/O, threads, format)
class readonly IOException extends Exception {
    // I/O failures (file, network, process, etc.)
}
class readonly FileNotFoundException extends IOException {
    // Path does not exist or is not a file
}
class readonly FormatException extends Exception {
    // Invalid string format (e.g. DateTime.parse, base64Decode)
}
class readonly InterruptedException extends Exception {
    // Thread interrupted while blocked (e.g. join, sleep)
}
```

All exceptions must extend `Exception`. Runtime exceptions (subclasses of `RuntimeException`) represent programming errors and do not require declaration in method signatures. Checked exceptions (direct subclasses of `Exception` or custom exceptions) must be declared using `throws`. The standard library uses the exceptions above; see [stdlib.md](stdlib.md) for which methods throw which exceptions.

### Declaring exceptions

Methods that can throw checked exceptions must declare them using the `throws` keyword after the method signature. Runtime exceptions do not require declaration.

#### Runtime exceptions (no declaration required)

Runtime exceptions represent programming errors and do not need to be declared. They can be thrown implicitly by operations or explicitly in code:

```nl
// No throws declaration needed - ArithmeticException is a RuntimeException
float divide(int a, int b) {
    return a / b; // May throw ArithmeticException if b is zero
}

// Explicit throw of runtime exception - still no throws needed
float divideWithCheck(int a, int b) {
    if (b == 0) {
        throw new ArithmeticException("Division by zero");
    }
    return a / b;
}

// Runtime exception handled internally - no throws needed
float safeDivide(int a, int b) {
    try {
        return a / b;
    }
    catch (ArithmeticException ex) {
        return 0.0; // Exception handled internally, does not propagate
    }
}
```

#### Checked exceptions (declaration required)

Checked exceptions must be declared using `throws`:

```nl
class Risky
{
    public void doSomethingRisky() throws Exception 
    {
        throw new Exception("You cannot do this");
    }
    
    public void doMultipleThings() throws MyException, AnotherException
    {
        if (condition) {
            throw new MyException("Error 1");
        }
        throw new AnotherException("Error 2");
    }
}
```

#### Generic Exception

```nl
class Risky
{
    public void doSomethingRisky() throws Exception 
    {
        throw new Exception("You cannot do this");
    }
}
```

#### Custom Exception

```nl
class readonly MyException extends Exception {
    public int customField;
    public construct(string message, int customField) {
        super(message);
        this.customField = customField;
    }
}

class Risky {
    public void doSomethingRisky() throws MyException {
        throw new MyException("You cannot do this", 42);
    }
}

class Test {
    public void do() {
        try {
            this.doSomethingRisky();
        }
        catch (MyException ex) {
            system.Out.print(ex.customField);
        }
    }
}
```

### Exception handling

When calling a method that declares `throws` for checked exceptions, the exception must be either:

1. Caught in a `try-catch` block
2. Propagated by declaring `throws` in the calling method

Runtime exceptions can be caught but do not require handling:

```nl
class Test {
    public void handleCheckedException() {
        try {
            this.doSomethingRisky(); // OK: checked exception is caught
        }
        catch (Exception ex) {
            system.Out.print(ex.message);
        }
    }
    
    // Error: doSomethingRisky() throws Exception but it's not handled
    // public void ignoreException() {
    //     this.doSomethingRisky();
    // }
    
    public void propagateException() throws Exception {
        this.doSomethingRisky(); // OK: exception is propagated
    }
    
    // Runtime exceptions do not require handling
    public void useRuntimeException() {
        float result = divide(10, 2); // OK: ArithmeticException may occur but no handling required
    }
    
    // Runtime exceptions can still be caught if desired
    public void catchRuntimeException() {
        try {
            float result = divide(10, 0);
        }
        catch (ArithmeticException ex) {
            system.Out.print("Division by zero handled");
        }
    }
}
```

#### Catch

```nl
try {
    doSomethingRisky();
}
catch (CustomException exception) {
    // specific catch
    system.Out.print(exception.message);
}
catch (Exception exception) {
    // generic exception
    system.Out.print(exception.message);
}
finally {
    // finally block
}
```

### Constructors with exceptions

Constructors can also declare `throws`. When creating an object with `new`, the exception must be handled:

```nl
class Resource {
    private system.io.FileHandle handle;
    
    public construct(string filename) throws FileNotFoundException {
        this.handle = system.io.File.open(filename); // May throw FileNotFoundException
    }
}

// Usage
try {
    auto resource = new Resource("data.txt");
}
catch (FileNotFoundException ex) {
    system.Out.print("File not found: " + ex.message);
}
```

### Exception inheritance rules

When overriding a method, the overriding method's `throws` clause must **cover** every checked exception declared by the parent (Liskov substitution principle):

- For each exception type `E` in the parent's `throws` clause, the child's `throws` clause must include `E` or a subclass of `E`.
- The child may not introduce new checked exception types that are not subtypes of exceptions declared by the parent.

Runtime exceptions are not considered in this rule.

```nl
class Base {
    public void doSomething() throws Exception, IOException {
        // ...
    }
}

class Derived extends Base {
    // OK: same exceptions
    public void doSomething() throws Exception, IOException {
        super.doSomething();
    }
    
    // OK: IOException is a subclass of Exception — covers both parent exceptions
    public void doSomething() throws IOException {
        super.doSomething();
    }
    
    // OK: subclasses (more specific)
    public void doSomething() throws FileNotFoundException, IOException {
        // FileNotFoundException extends IOException
        super.doSomething();
    }
    
    // Error (E016): child does not cover IOException — Exception is not a subclass of IOException
    // public void doSomething() throws Exception {
    //     super.doSomething();
    // }
    
    // OK: can throw runtime exceptions without declaring them
    public void doSomething() throws Exception, IOException {
        int[] arr = new int[10];
        arr[20] = 5; // May throw IndexOutOfBoundsException, but no declaration needed
        super.doSomething();
    }
}

// E017: introducing an exception not in the parent's hierarchy
class BaseNarrow {
    public void doSomething() throws IOException { /* ... */ }
}
class DerivedNarrow extends BaseNarrow {
    // OK: same as parent
    public void doSomething() throws IOException {
        super.doSomething();
    }
    // Error (E017): SomeOtherException is not a subtype of IOException
    // public void doSomething() throws IOException, SomeOtherException {
    //     super.doSomething();
    // }
}
```

This rule ensures substitutability: callers that handle the parent's declared exceptions can safely call the overriding method, since the child only throws the same or more specific exceptions. Runtime exceptions do not need to be declared in overridden methods.

## Standard library

System interaction (console, parsing, file system, network, threads, time, environment, process, text utilities) is provided by the **standard library** in the namespaces `system`, `system.io`, `system.net`, `system.thread`, `system.time`, `system.Env`, `system.ps`, and `system.text`. Types such as `system.Out`, `system.Err`, `system.In`, `system.Int`, `system.Float`, `system.String`, `system.io.File`, and `system.io.FileHandle` offer static or instance methods for I/O and parsing. A `system.io.FileHandle` (from `system.io.File.open`) supports `read`, `readLine`, `write`, and `flush` for line-by-line or binary access in addition to `close`. See **[stdlib.md](stdlib.md)** for the full API.

## Entry point

The entry point of an NL program is defined by a `main` method, similar to Java. The method must be declared as `public static` and must return an `int` value, which serves as the program's exit code. The exit code is used by the operating system to determine whether the program executed successfully (typically 0 for success, non-zero for errors).

### Method signature

The `main` method must have the following exact signature:

```nl
public static int main(string[] args)
```

- **`args`**: An array of strings containing the command-line arguments. The first element (`args[0]`) is the program name, followed by any additional arguments provided when executing the program. Use `args.length()` to get the number of arguments.
- **Return value**: An integer representing the exit code. A return value of `0` typically indicates successful execution, while non-zero values indicate various error conditions

### Class requirements

The `main` method can be defined in any class within the program. The class name is not restricted and does not need to be named `Program`. However, there must be exactly one `main` method in the entire program.

```nl
namespace com.example.project;

class Application {
    public static int main(string[] args) {
        system.Out.print("Application started");
        return 0;
    }
}
```

### Compilation requirements

A program must contain exactly one `main` method. If no `main` method is found during compilation, the compiler will emit an error. This requirement does not apply to library projects, which are intended to be used by other programs rather than executed directly.

### Usage example

```nl
namespace com.example.project;

class Calculator {
    public static int main(string[] args) {
        if (args.length() < 4) {
            system.Out.print("Usage: calculator <operation> <a> <b>");
            return 1; // Exit code 1 indicates an error
        }
        
        string operation = args[1];
        int a = system.Int.parseInt(args[2]);
        int b = system.Int.parseInt(args[3]);
        
        int result = 0;
        if (operation == "add") {
            result = a + b;
        } else if (operation == "multiply") {
            result = a * b;
        } else {
            system.Out.print("Unknown operation: " + operation);
            return 1;
        }
        
        system.Out.print("Result: " + result);
        return 0; // Exit code 0 indicates success
    }
}
```

When executed, the return value of the `main` method becomes the program's exit code, which can be used by shell scripts, build systems, and other programs to determine the execution status.

## Planned

The following features may be added to the spec in future versions:

- **`char` type** — A dedicated scalar type for a single character (e.g. Unicode codepoint). Currently, a character is represented as a `string` of length 1.

- **RAII / try-with-resources** — No mechanism for automatic resource cleanup at scope exit (like Java's `try-with-resources`, C#'s `using`, or C++ RAII) is specified for now. The only cleanup mechanism is the destructor, whose timing is implementation-defined. A dedicated construct may be considered in a future version.


[1]: #conditionals

[2]: #loops

[3]: #switchmatch

[4]: #auto-type

[5]: #class-methods

[6]: #native-types

[7]: #null-initialization-and-default-values

[8]: #source-code-files

[9]: #extends-implements

[10]: #self-and-type-keywords

[11]: #readonly

[12]: #template-class

[13]: #operator-overloading

[14]: #exception-handling

[15]: #declaring-exceptions

[16]: #constructors-and-destructors

[17]: #enums

[18]: #classes

[19]: #visibility

[20]: #nodiscard

[21]: #super-keyword

[22]: #imports

[23]: #parameter-passing-semantics

[24]: #final-classes-and-methods

[25]: #abstract-classes-and-methods