---
description: Architectural reference for AllLegacyLivingEntityTypesQuery
---

# AllLegacyLivingEntityTypesQuery

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** Singleton

## Definition
```java
// Signature
@Deprecated
public class AllLegacyLivingEntityTypesQuery implements Query<EntityStore> {
```

## Architecture & Concepts

The AllLegacyLivingEntityTypesQuery is a specialized filter component within the server-side Entity Component System (ECS). Its sole function is to identify entity archetypes that conform to a legacy definition of a "living entity".

This class acts as a predicate in ECS queries. When a system needs to operate on all entities considered "living" by older game logic, it uses this query to select the appropriate archetypes. The actual determination logic is delegated to the EntityUtils helper class, making this query a simple, stateless adapter.

**WARNING:** This class is deprecated. It exists for backward compatibility with core systems that have not been migrated to more explicit component-based queries. New systems should **never** rely on this query. Instead, they should query for specific components that define an entity's capabilities, such as HealthComponent or AIComponent.

## Lifecycle & Ownership

-   **Creation:** The single instance of this class is created by the Java ClassLoader when the AllLegacyLivingEntityTypesQuery class is first referenced. It is exposed via the public static final field named INSTANCE.
-   **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the server process.
-   **Destruction:** The object is garbage collected by the JVM when the server process terminates. There is no manual destruction logic.

## Internal State & Concurrency

-   **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is entirely deterministic based on the input archetype.
-   **Thread Safety:** This class is inherently **thread-safe**. As a stateless singleton, it can be passed to and executed by multiple threads concurrently without any risk of race conditions or need for external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | AllLegacyLivingEntityTypesQuery | O(1) | The canonical singleton instance of the query. |
| test(archetype) | boolean | O(C) | Returns true if the provided archetype matches the legacy definition of a living entity. C is the number of components in the archetype. |
| requiresComponentType(componentType) | boolean | O(1) | Always returns false. This query does not use component pre-filtering, which may impact performance in large-scale queries. |
| validateRegistry(registry) | void | O(1) | No-op. This query has no dependencies to validate. |
| validate() | void | O(1) | No-op. This query has no internal state to validate. |

## Integration Patterns

### Standard Usage

This query is intended to be used with an ECS manager to retrieve a set of archetypes that can then be used to iterate over specific entities.

```java
// Correctly retrieve and use the singleton instance for a query
EntitySystem entitySystem = server.getWorld().getEntitySystem();
Set<Archetype> livingArchetypes = entitySystem.queryArchetypes(AllLegacyLivingEntityTypesQuery.INSTANCE);

for (Archetype archetype : livingArchetypes) {
    // Perform operations on all entities matching this legacy archetype
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new AllLegacyLivingEntityTypesQuery()`. This creates a redundant object and violates the singleton pattern. Always use the static `INSTANCE` field.
-   **Usage in New Systems:** Do not use this query for any new game feature development. Its definition of "living" is opaque and tied to legacy code. New features must use explicit component queries (e.g., `Query.Builder.all(HealthComponent.class, PhysicsComponent.class).build()`).

## Data Pipeline

This class acts as a filter predicate within a larger data flow. It does not transform data itself.

> Flow:
> Entity System Query Request -> Query Executor -> **AllLegacyLivingEntityTypesQuery.test(archetype)** -> Filtered Set of Archetypes -> Game Logic System

