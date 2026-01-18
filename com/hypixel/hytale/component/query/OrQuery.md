---
description: Architectural reference for OrQuery
---

# OrQuery

**Package:** com.hypixel.hytale.component.query
**Type:** Transient

## Definition
```java
// Signature
public class OrQuery<ECS_TYPE> implements Query<ECS_TYPE> {
```

## Architecture & Concepts
The OrQuery class is a structural component within the Entity Component System (ECS) query framework. It embodies the Composite design pattern, allowing multiple individual Query objects to be treated as a single, unified Query. Its primary function is to implement a logical OR operation for entity selection.

This component acts as a fundamental building block for constructing complex queries. It enables systems to filter for entities that match *at least one* of a given set of criteria. For example, a system could use an OrQuery to find all entities that are either *on fire* or *poisoned*, where each condition is represented by a separate sub-query.

OrQuery operates at the query definition layer. It defines the logical structure of a filter but does not participate in the low-level iteration or data access of the matching entities. The actual evaluation is performed against an Archetype, which represents a unique combination of component types.

## Lifecycle & Ownership
- **Creation:** An OrQuery is instantiated directly by client code, typically within a system's initialization phase when defining the set of entities it will operate on. It is not managed by a service locator or dependency injection framework.
- **Scope:** The lifetime of an OrQuery instance is bound to the system or query definition that created it. In most cases, it is a short-lived object, constructed and used for the duration of a single query execution. If a system caches its queries, the OrQuery will persist as part of that cached structure.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as it is no longer referenced. There are no manual cleanup or disposal requirements.

## Internal State & Concurrency
- **State:** The internal state consists of a final array of sub-queries. This makes the OrQuery instance itself **effectively immutable**. Its state is defined entirely at construction and cannot be modified thereafter.
- **Thread Safety:** The class is inherently **thread-safe for read operations**. All of its methods iterate over the internal, immutable array without performing any writes. Concurrency safety of a query execution therefore depends on the thread safety of the sub-queries it contains and the underlying ECS data structures, not on the OrQuery itself.

## API Surface
The public contract of OrQuery is an implementation of the Query interface. Its methods delegate the operation to its collection of sub-queries.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Archetype) | boolean | O(N) | Evaluates the query against an Archetype. Returns true if **any** sub-query returns true. N is the number of sub-queries. |
| requiresComponentType(ComponentType) | boolean | O(N) | Checks if **any** sub-query requires the specified component type. |
| validateRegistry(ComponentRegistry) | void | O(N) | Propagates validation to all sub-queries. Throws if a sub-query is invalid. |
| validate() | void | O(N) | Propagates validation to all sub-queries. Throws if a sub-query is invalid. |

## Integration Patterns

### Standard Usage
OrQuery is used to combine two or more queries. The most common pattern is to compose simple queries, such as Has, into a more expressive filter.

```java
// Assume Has is a Query implementation that checks for a component's existence.
Query<MyECSType> hasHealth = new Has<>(HealthComponent.class);
Query<MyECSType> hasMana = new Has<>(ManaComponent.class);

// Create a query that finds entities with EITHER a HealthComponent OR a ManaComponent.
Query<MyECSType> hasHealthOrMana = new OrQuery<>(hasHealth, hasMana);

// This composite query can now be used by an entity processing system.
world.query(hasHealthOrMana).forEach(entity -> {
    // Process entities with health or mana...
});
```

### Anti-Patterns (Do NOT do this)
- **Empty Instantiation:** Do not create an OrQuery with no arguments (`new OrQuery<>()`). This query is valid but will always evaluate to false, resulting in dead code or incorrect logic.
- **Redundant Nesting:** Avoid wrapping a single query in an OrQuery (`new OrQuery<>(someQuery)`). This adds a needless layer of indirection and offers no functional benefit over using `someQuery` directly.
- **Manual Deep Nesting:** While functionally correct, manually nesting OrQuery instances (`new OrQuery(q1, new OrQuery(q2, q3))`) is less clear and potentially less performant than a flat structure (`new OrQuery(q1, q2, q3)`). High-level query builders should automatically flatten such structures.

## Data Pipeline
OrQuery functions as a predicate within the ECS query pipeline. It does not transform data but instead filters the set of Archetypes that are considered for entity processing.

> Flow:
> System Request -> Query Builder -> **OrQuery (Composition)** -> Query Execution Engine -> Archetype Matching -> Filtered Entity Set

