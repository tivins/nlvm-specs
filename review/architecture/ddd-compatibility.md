# NL and Domain-Driven Design — Compatibility Analysis

This document analyzes whether projects using NLVM can adhere to Domain-Driven Design (DDD) principles. It is based on the NL language specifications (specs.md, compiler.md, stdlib.md, vm.md) and coherence tracking (review/coherence.md).

---

## Overall Verdict

**Yes, NLVM projects can adhere to DDD principles.** The NL language provides the essential building blocks. Some aspects require conventions or patterns to be implemented manually.

---

## Strengths for DDD

| DDD Principle | NL Support | Details |
|---------------|------------|---------|
| **Value Objects** | Very good | `readonly` (class or property), `ValueEquatable`, `const` methods/parameters. Immutability and structural equality well supported. |
| **Entities** | Good | Classes with identity (reference), constructors, encapsulation via `private`/`protected`. |
| **Aggregates** | Good | Encapsulation, invariants in constructors. No native notion of "aggregate root", but it is a pattern, not a language constraint. |
| **Domain Services** | Good | Stateless classes or `static` methods. |
| **Repositories** | Good | Interfaces to abstract persistence. E.g.: `interface IOrderRepository { Order|null findById(int id); void save(Order o); }`. |
| **Bounded Contexts** | Good | Namespaces aligned with folder hierarchy (`namespace com.example.order.domain`). One file = one class. |
| **Ubiquitous Language** | Good | `typedef` for domain types, explicit naming. |
| **Infrastructure isolation** | Good | The domain depends on interfaces; infrastructure implements them. |
| **Application Services** | Good | Classes that orchestrate use cases. |

---

## Aspects to Handle by Convention or Implementation

| Aspect | Situation | Recommendation |
|--------|-----------|----------------|
| **Dependency Injection** | No built-in DI | Manual injection via constructors. An entry point (composition root) builds dependencies and passes them to services. |
| **Domain Events** | No dedicated mechanism | Implement a simple event bus (subscriber list, synchronous dispatch) or a similar pattern. |
| **Multiple interfaces** | VM supports multiple interfaces, syntax not clearly documented | Coherence mentioned in review/coherence.md (II-7). In practice, syntax like `implements A, B, C` is likely. |
| **Access control between modules** | No notion of module with visibility | Organize by namespaces and conventions (e.g. `domain` does not depend on `infrastructure`). |

---

## Example DDD Structure in NL

```nl
// domain/Order.nl — Entity
namespace com.example.order.domain;
class Order {
    private int id;
    private OrderStatus status;
    public construct(int id, OrderStatus status) { ... }
}

// domain/OrderStatus.nl — Value Object (enum)
namespace com.example.order.domain;
enum OrderStatus { Pending, Confirmed, Shipped }

// domain/IOrderRepository.nl — Port (interface)
namespace com.example.order.domain;
interface IOrderRepository {
    public Order|null findById(int id);
    public void save(const Order order);
}

// application/PlaceOrderService.nl — Application Service
namespace com.example.order.application;
use com.example.order.domain.*;
class PlaceOrderService {
    private IOrderRepository repo;
    public construct(IOrderRepository repo) { this.repo = repo; }
    public void execute(PlaceOrderCommand cmd) { ... }
}

// infrastructure/FileOrderRepository.nl — Adapter
namespace com.example.order.infrastructure;
use com.example.order.domain.*;
class FileOrderRepository implements IOrderRepository { ... }
```

---

## Summary

| Criterion | Score | Comment |
|-----------|-------|---------|
| Domain modeling | 9/10 | Value objects, entities, enums, interfaces. |
| Layer separation | 8/10 | Interfaces + namespaces sufficient. |
| Dependency inversion | 7/10 | Possible via constructors, without framework. |
| Bounded contexts | 8/10 | Namespaces suitable. |
| Domain events | 5/10 | To be implemented manually. |

---

## Conclusion

NL is suited for DDD. Projects will need to define conventions (layers, composition root, possibly a lightweight event bus), but the language offers the necessary foundations. The absence of built-in DI is not blocking: manual constructor injection is a common approach in DDD.

<hr>

## Detailed Explanation

### Value Objects

In DDD, a Value Object is defined by its attributes, not by an identity. Two Value Objects with the same values are interchangeable. NL supports this pattern in several ways:

- **`readonly` on a class:** all properties become immutable after construction. No modification is possible once the object is created.
- **`readonly` on a property:** only that property is frozen after the constructor.
- **`ValueEquatable`:** the interface requires `valueEquals(const Self|null other)` and `valueHash()`. Structural equality (same values) replaces reference equality. Ideal for `system.Map` keys based on value.
- **`const`:** on methods (no mutation of the object) and parameters (no modification of the parameter). Ensures that a Value Object passed as an argument will not be altered.

Typed enums (`enum OrderStatus : string`) also constitute lightweight Value Objects for closed sets of values.

### Entities and Identity

A DDD Entity has a stable identity that distinguishes it from others, even when its attributes change. In NL, class instances have identity by reference: `a == b` compares references, not values. The Entity therefore naturally retains its identity across mutations. Encapsulation (`private`, `protected`) protects state and enforces invariants in public methods.

### Aggregates and Aggregate Roots

An Aggregate is a cluster of domain objects whose consistency boundaries are guaranteed by an Aggregate Root. NL does not provide a dedicated concept, but the pattern is achievable:

- The Aggregate Root is a class that exposes modification operations.
- Internal entities are `private` or `protected`; only the Root's methods allow access.
- Invariants are checked in the constructor and in methods that modify state.
- The Root can be `final` to prevent subclasses that would bypass invariants.

### Repositories and Ports/Adapters

The Repository pattern abstracts persistence behind an interface. In NL, an interface is defined in the domain (port) and a concrete implementation in infrastructure (adapter). The domain depends only on the interface; infrastructure depends on the domain and implements the interface. Dependency inversion is thus respected without a framework.

The Application Service receives the Repository via its constructor. The entry point (`main`) or a factory builds the concrete implementations and injects them. This is the "composition root" pattern.

### Bounded Contexts and Namespaces

A DDD Bounded Context delimits a coherent model and ubiquitous language. In NL, namespaces reflect the directory hierarchy: `com.example.order.domain`, `com.example.inventory.domain`, etc. Each context can have its own submodel: `order.domain`, `order.application`, `order.infrastructure`. Imports (`use`) make dependencies between contexts explicit. A strict convention (e.g. "the domain never depends on application or infrastructure") can be enforced by code review or analysis tools.

### Ubiquitous Language

The ubiquitous language is the vocabulary shared between business experts and developers. In NL, `typedef` allows naming domain types:

```nl
typedef string Email;
typedef int CustomerId;
```

Class, method, and property names reflect the business. Enums capture explicit business states or values.

### Domain Events

Domain Events signal that a business fact has occurred. NL has no native mechanism (event type, event bus). A manual implementation consists of:

- Defining an `IDomainEvent` interface or a base event class.
- Creating a handler registry (list of callbacks or objects).
- Dispatching events synchronously after a domain operation.

Closures and interfaces allow passing handlers without difficulty.

### What NL Does Not Provide (and How to Compensate)

- **DI:** no container or automatic injection. The composition root is manual.
- **Module-level visibility:** no "package private" or modules with explicit visibility. Conventions and team discipline compensate.
- **RAII / try-with-resources:** resource cleanup relies on destructors (timing not guaranteed) or `try/finally` blocks. An explicit release pattern can be defined in the domain.

### References to Specifications

The points above are based on: specs.md (§ Classes, § Visibility, § Readonly, § ValueEquatable, § Extends/Implements, § Imports), stdlib.md (§ Core interfaces), compiler.md (visibility and type checks), vm.md (module format and object representation).
