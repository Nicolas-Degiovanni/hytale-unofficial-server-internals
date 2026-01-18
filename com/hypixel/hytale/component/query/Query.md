---
description: Architectural reference for Query
---

# Query<ECS_TYPE>

**Package:** com.hypixel.hytale.component.query
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface Query<ECS_TYPE> {
```

## Architecture & Concepts
The Query interface is the cornerstone of the Entity-Component-System (ECS) filtering mechanism. It defines a formal contract for creating predicates that test whether an entity, represented by its Archetype, matches a specific set of component criteria. This allows systems to efficiently retrieve collections of entities they need to operate on without iterating through every entity in the world.

Architecturally, this interface and its associated concrete classes (AndQuery, OrQuery, NotQuery) implement the **Composite design pattern**. Simple queries can be combined into complex, tree-like structures using logical operators. The static factory methods, such as **and** and **or**, serve as the primary construction mechanism for building these query trees.

The Query is not the system that finds entities; it is the *specification* of what to find. The core game loop or a world manager will take a Query object and use it to filter entities, passing each entity's Archetype to the **test** method.

## Lifecycle & Ownership
- **Creation:** Query objects are lightweight and transient. They are intended to be created on-demand using the static factory methods provided on the interface, for example, `Query.and(...)`. They are not managed by any service locator or dependency injection framework.

- **Scope:** The lifetime of a Query is typically scoped to a single method execution. A game system will construct a Query, use it to fetch entities for a single game tick, and then the Query object becomes eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Standard implementations like AndQuery and OrQuery are effectively immutable; they store a final collection of sub-queries provided at construction time. They do not cache results or modify their internal state during the **test** operation.

- **Thread Safety:** The standard implementations are inherently thread-safe for read operations like **test** due to their immutable nature. Multiple threads can safely use the same Query instance to test different Archetypes concurrently. However, the validation methods (**validateRegistry** and **validate**) may perform internal initialization and are **not** guaranteed to be thread-safe.

**WARNING:** The validation methods should be called from a single, controlled thread during an initialization phase before the Query is used in a multithreaded context.

## API Surface
The public contract is dominated by static factory methods for composition and the primary **test** method for evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| any() | static AnyQuery | O(1) | Creates a query that matches all archetypes. |
| not(Query) | static NotQuery | O(1) | Creates a query that inverts the result of the provided query. |
| and(Query...) | static AndQuery | O(1) | Creates a query that matches only if all sub-queries match. |
| or(Query...) | static OrQuery | O(1) | Creates a query that matches if any sub-query matches. |
| test(Archetype) | boolean | O(N) | Evaluates the query against an archetype. N is the number of sub-queries. |
| validateRegistry(...) | void | O(N) | Validates that all components within the query exist in the registry. |

## Integration Patterns

### Standard Usage
The primary pattern is to compose queries using the static factories and pass the resulting object to an entity management system.

```java
// Example: Find all entities that have a Position and Velocity component,
// but do NOT have a Stunned component.

Query query = Query.and(
    new HasComponentQuery(Position.class),
    new HasComponentQuery(Velocity.class),
    Query.not(new HasComponentQuery(Stunned.class))
);

// The world or entity manager then uses this query
EntitySet matchingEntities = world.getEntities(query);
for (Entity entity : matchingEntities) {
    // Process entity
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Implementation:** Avoid creating a custom class that implements the Query interface directly. The provided composition logic (**and**, **or**, **not**) is heavily optimized and covers nearly all use cases. Direct implementation risks breaking internal engine assumptions.

- **Long-Term Storage:** Do not store Query objects in long-lived fields. They are designed to be cheap to create and should be constructed as needed. Storing them can prevent component types from being garbage collected and may lead to subtle bugs if the game's component structure changes.

- **Validation After Use:** The **validate** and **validateRegistry** methods must be called before the query is used in performance-critical loops. Failing to do so can result in runtime exceptions or unpredictable behavior during the **test** phase.

## Data Pipeline
The Query acts as a filter specification within a larger data retrieval flow. It does not transform data itself but dictates which data is selected.

> Flow:
> Game System Logic -> **Query Composition (e.g., Query.and)** -> WorldManager.getEntities(query) -> Internal Archetype Iteration -> **query.test(archetype)** -> Filtered Entity Collection -> Game System Logic

