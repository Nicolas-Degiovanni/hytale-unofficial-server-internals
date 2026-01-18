---
description: Architectural reference for AndQuery
---

# AndQuery

**Package:** com.hypixel.hytale.component.query
**Type:** Transient

## Definition
```java
// Signature
public class AndQuery<ECS_TYPE> implements Query<ECS_TYPE> {
```

## Architecture & Concepts
The AndQuery class is a structural component within the Entity-Component-System (ECS) query framework. It implements the Composite design pattern to combine multiple distinct Query objects into a single, logical unit. Its primary function is to act as a logical **AND** operator.

When an AndQuery is evaluated against an entity Archetype, it will only return true if **all** of its constituent sub-queries also return true. This mechanism is fundamental for creating highly specific entity selectors. For example, a system that handles physics might construct a query to find entities that have *both* a PositionComponent *and* a VelocityComponent *and* a RigidBodyComponent. AndQuery is the component that enables this conjunctive filtering.

It does not perform any filtering itself; it delegates the test operation to its children and aggregates the results according to its AND logic.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, typically by an EntitySystem or a query builder utility during its own initialization phase. It is a lightweight, value-like object.
- **Scope:** The lifetime of an AndQuery instance is bound to the object that created it. If it is part of a system's static configuration, it will persist for the lifetime of that system. If created for a one-off search, it will be garbage collected after the operation completes.
- **Destruction:** Managed entirely by the Java garbage collector. No explicit cleanup or resource release is required.

## Internal State & Concurrency
- **State:** The AndQuery is effectively **immutable** after construction. The internal array of sub-queries is final and is not modified post-instantiation. Its behavior is determined solely by its initial configuration.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state. It can be safely shared and executed by multiple threads simultaneously.
    - **WARNING:** While the AndQuery itself is thread-safe, the overall thread safety of an evaluation depends on the sub-queries it contains. Ensure that all composed Query objects are also thread-safe if concurrent execution is a requirement.

## API Surface
The public contract of AndQuery is focused on fulfilling the Query interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Archetype) | boolean | O(N) | Evaluates the given Archetype. Returns true only if all sub-queries pass. Short-circuits and returns false on the first failure. N is the number of sub-queries. |
| requiresComponentType(ComponentType) | boolean | O(N) | Checks if any of the contained sub-queries depend on the specified ComponentType. Used for query optimization and validation. |
| validateRegistry(ComponentRegistry) | void | O(N) | Propagates the validation call to all sub-queries, ensuring each is valid against the given registry. |
| validate() | void | O(N) | Propagates a context-free validation call to all sub-queries. |

## Integration Patterns

### Standard Usage
AndQuery is used to compose multiple conditions. The most common pattern is to instantiate it with a set of other Query implementations to define the precise data layout an EntitySystem operates on.

```java
// Define a query for entities that have both position and velocity
Query<GameEntity> physicsQuery = new AndQuery<>(
    new HasComponentQuery<>(POSITION_COMPONENT),
    new HasComponentQuery<>(VELOCITY_COMPONENT)
);

// This query can now be used by a system to filter for relevant entities
entityWorld.forEach(physicsQuery, (entity) -> {
    // Process physics for the entity
});
```

### Anti-Patterns (Do NOT do this)
- **Empty Instantiation:** Do not create an AndQuery with no arguments (`new AndQuery<>()`). This creates a query that always returns true, which can lead to systems incorrectly processing every entity in the world, causing severe performance degradation and logical errors.
- **Redundant Nesting:** Avoid nesting AndQuery instances within each other, such as `new AndQuery(q1, new AndQuery(q2, q3))`. While functionally correct, this creates unnecessary object allocations and adds depth to the call stack. The constructor accepts a variable number of arguments, so a flattened structure `new AndQuery(q1, q2, q3)` is always preferred for clarity and performance.

## Data Pipeline
AndQuery acts as a predicate or filter within the ECS data-retrieval pipeline. It does not transform data but determines which data streams (Archetypes) are allowed to proceed for processing.

> Flow:
> EntitySystem requests entities -> QueryExecutor iterates Archetypes -> **AndQuery.test(archetype)** -> Archetype is accepted or rejected -> System receives and processes only accepted entities.

