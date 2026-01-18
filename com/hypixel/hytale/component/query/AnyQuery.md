---
description: Architectural reference for AnyQuery
---

# AnyQuery

**Package:** com.hypixel.hytale.component.query
**Type:** Singleton

## Definition
```java
// Signature
class AnyQuery<ECS_TYPE> implements Query<ECS_TYPE> {
```

## Architecture & Concepts
The AnyQuery class is a fundamental component of the Entity-Component-System (ECS) query engine. It represents a universal, non-filtering query that matches all possible entity archetypes. Its primary role is to serve as a "wildcard" or "match-all" predicate within the query system.

Unlike more specific queries that test for the presence or absence of certain components, AnyQuery's implementation of the `test` method unconditionally returns true. This makes it an efficient mechanism for systems that need to iterate over every single entity in a world, regardless of its composition. It acts as the base case for query composition and is often used internally by higher-level ECS managers to fetch a complete set of entities before applying further filtering.

Because its behavior is constant and it holds no state, it is implemented as a static singleton instance to prevent unnecessary object allocation.

### Lifecycle & Ownership
- **Creation:** The single `INSTANCE` is instantiated by the JVM during class loading. Its lifecycle is not managed by any framework or dependency injection container; it exists as a static constant.
- **Scope:** Application-wide. The singleton instance persists for the entire lifetime of the application.
- **Destruction:** The instance is eligible for garbage collection only when its class loader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** AnyQuery is stateless and immutable. It contains no instance fields and its behavior is predetermined at compile time.
- **Thread Safety:** This class is inherently thread-safe. As a stateless singleton, it can be safely accessed and used by any number of threads simultaneously without locks or other synchronization primitives.

## API Surface
The public contract is defined by the Query interface. The key implementation detail is the unconditional logic of its methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(archetype) | boolean | O(1) | Unconditionally returns true, matching any provided archetype. |
| requiresComponentType(componentType) | boolean | O(1) | Unconditionally returns false, as it has no component requirements. |
| validateRegistry(registry) | void | O(1) | Performs no operation. This query is valid in any context. |
| validate() | void | O(1) | Performs no operation. |

## Integration Patterns

### Standard Usage
The AnyQuery is not intended for direct use in most game logic. Instead, it is typically accessed via its static `INSTANCE` field by internal ECS systems or query builders that require a universal selector.

```java
// Retrieve the singleton instance for use in a query system
Query query = AnyQuery.INSTANCE;

// This will always evaluate to true
boolean matches = query.test(anyArchetype);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class is package-private, but even within the package, `new AnyQuery()` should never be used. This bypasses the singleton pattern, creating unnecessary objects and defeating the performance goal of a shared, static instance.
- **Subclassing:** There is no valid reason to extend AnyQuery. Its purpose is atomic and its behavior is complete. Extending it would suggest a misunderstanding of its role as a universal constant.

## Data Pipeline
AnyQuery functions as a predicate within the ECS data-access pipeline. It does not transform data but rather acts as a gate that is always open.

> Flow:
> ECS System Request -> Query Executor -> **AnyQuery.test(archetype)** -> Always returns `true` -> Archetype is included in the result set

