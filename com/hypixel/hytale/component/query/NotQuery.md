---
description: Architectural reference for NotQuery
---

# NotQuery

**Package:** com.hypixel.hytale.component.query
**Type:** Transient

## Definition
```java
// Signature
public class NotQuery<ECS_TYPE> implements Query<ECS_TYPE> {
```

## Architecture & Concepts
The NotQuery class is a structural component within the Entity-Component-System (ECS) query framework. It functions as a logical operator, implementing the Decorator pattern to provide negation for other query objects. Its primary role is to invert the test result of a wrapped Query instance.

This component is essential for constructing complex, exclusionary queries. For example, it enables developers to find all entities that match a certain criteria *except* for those that also possess a specific component. In a composite query tree, NotQuery acts as a unary operator node that modifies the behavior of its single child. It does not perform any direct component checks itself; it delegates all operations to the wrapped query and inverts the final boolean test.

## Lifecycle & Ownership
- **Creation:** NotQuery instances are created internally by higher-level query construction mechanisms, such as a fluent QueryBuilder. They are not intended to be instantiated directly by application-level code. A new NotQuery is created whenever a logical NOT operation is added to a composite query definition.
- **Scope:** The lifetime of a NotQuery object is ephemeral and strictly bound to the parent Query object that contains it. It exists only for the duration of the query's construction and subsequent evaluation.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the composite query it belongs to is no longer in use. No manual resource management is required.

## Internal State & Concurrency
- **State:** **Immutable**. The NotQuery class is stateless. Its single field, which holds a reference to the wrapped query, is declared final and is initialized at construction. The object's behavior is fixed for its entire lifetime.
- **Thread Safety:** **Conditionally Thread-Safe**. The NotQuery class itself introduces no concurrency hazards as it contains no mutable state. However, its overall thread safety is entirely dependent on the thread safety of the wrapped Query object it delegates to. If the wrapped query is thread-safe, then any NotQuery instance containing it will also be safe for concurrent use.

## API Surface
The public API is minimal, primarily consisting of the methods required to fulfill the Query interface contract. All operations are delegated to the wrapped query.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Archetype) | boolean | O(wrapped) | The core function. Executes the test method on the wrapped query and returns the logical inverse of the result. |
| requiresComponentType(ComponentType) | boolean | O(wrapped) | Delegates to the wrapped query to determine if a specific component type is required for evaluation. |
| validateRegistry(ComponentRegistry) | void | O(wrapped) | Delegates the validation logic against a given ComponentRegistry to the wrapped query. |
| validate() | void | O(wrapped) | Delegates the internal validation logic to the wrapped query. |

## Integration Patterns

### Standard Usage
NotQuery should not be used directly. It is an implementation detail of the query system, typically invoked via a builder or a static factory method.

```java
// CORRECT: Using a hypothetical QueryBuilder to construct a complex query.
// The builder is responsible for creating the NotQuery instance internally.

Query<GameEntity> hasVelocity = QueryBuilder.create().with(Velocity.class).build();
Query<GameEntity> hasPhysics = QueryBuilder.create().with(PhysicsBody.class).build();

// Find entities with a physics body but NOT a velocity component.
Query<GameEntity> finalQuery = QueryBuilder.create()
    .all(hasPhysics)
    .none(hasVelocity) // This would internally create a NotQuery
    .build();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class directly with the new keyword. The query system relies on its own construction logic to build a valid and optimized query tree. Direct creation bypasses this process.
- **Null Wrapping:** Constructing a NotQuery with a null inner query will lead to a NullPointerException during any subsequent method call. The query building framework should prevent this, but direct instantiation exposes this risk.

## Data Pipeline
The NotQuery component acts as a simple transformation step in the query evaluation pipeline. It does not source or sink data, but rather modifies the boolean logic flow.

> Flow:
> Query Executor -> **NotQuery.test(archetype)** -> WrappedQuery.test(archetype) -> `boolean` result -> Logical NOT operation -> Final `boolean` result

