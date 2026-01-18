---
description: Architectural reference for ExactArchetypeQuery
---

# ExactArchetypeQuery

**Package:** com.hypixel.hytale.component.query
**Type:** Transient

## Definition
```java
// Signature
public class ExactArchetypeQuery<ECS_TYPE> implements Query<ECS_TYPE> {
```

## Architecture & Concepts
The ExactArchetypeQuery is a concrete implementation of the Query interface within Hytale's Entity-Component-System (ECS) framework. Its purpose is to provide a highly specific and performant mechanism for identifying entities that match a precise component layout.

Unlike more flexible queries that might search for entities *containing* a set of components, this class filters for entities whose component signature is an **exact match** to a given Archetype. If an entity has even one component more or less than the target Archetype, the query's test method will return false.

This strictness makes it an essential tool for systems that must operate on a well-defined, immutable set of components. It acts as a filtering strategy, passed to higher-level entity managers to retrieve a specific cohort of entities from the game world.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor (`new ExactArchetypeQuery(archetype)`). It is typically created by a System or a service that needs to perform a query. It is not managed by a dependency injection container or a central registry.
- **Scope:** The object's lifetime is intended to be short. It is often created, used for a single query operation within a game tick, and then becomes eligible for garbage collection. Systems that repeatedly query for the same archetype may cache an instance for their own lifetime.
- **Destruction:** Managed entirely by the Java garbage collector. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** Immutable. The internal Archetype reference is declared final and is set only once during construction. The object's state cannot be modified after it is created.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. A single instance can be safely passed to and used by multiple threads simultaneously without any external locking or synchronization.

## API Surface
The public contract is minimal, focused on fulfilling the Query interface and delegating logic to the contained Archetype.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(archetype) | boolean | O(1) | The core filtering logic. Returns true only if the provided Archetype is an exact structural match to the one held by the query. |
| requiresComponentType(componentType) | boolean | O(N) | Delegates to the internal Archetype to determine if it includes the specified component type. N is the number of components in the archetype. |
| validateRegistry(registry) | void | - | A pass-through validation call. Delegates to the internal Archetype to ensure all its component types are known to the provided ComponentRegistry. |
| validate() | void | - | A pass-through validation call. Delegates to the internal Archetype to perform self-consistency checks. |
| getArchetype() | Archetype | O(1) | Returns the target Archetype this query is configured to match. |

## Integration Patterns

### Standard Usage
The primary pattern is to construct an Archetype representing the exact entity structure you wish to find, create an ExactArchetypeQuery with it, and pass it to an entity management system.

```java
// 1. Define the exact Archetype for entities that can be moved by physics.
Archetype<GameEntity> physicsBodyArchetype = Archetype.newBuilder(GameEntity.class)
    .with(Position.class)
    .with(Velocity.class)
    .with(Mass.class)
    .build();

// 2. Create a query object for this specific Archetype.
Query<GameEntity> exactPhysicsQuery = new ExactArchetypeQuery<>(physicsBodyArchetype);

// 3. Pass the query to the world to get a filtered list of entities.
//    This will NOT match entities that also have a Health component, for example.
QueryResult<GameEntity> results = world.query(exactPhysicsQuery);

for (Entity entity : results.getEntities()) {
    // Process entities that match exactly.
}
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Granularity:** Do not use this query when you need to find entities that *at least have* a set of components but may have others. This query is for exact matches only. Using it to find "all entities with a Position" will fail if those entities also have other components. A different Query implementation must be used for such cases.
- **Instantiation in Hot Loops:** While instantiation is cheap, avoid creating a new ExactArchetypeQuery for the same archetype on every iteration of a performance-critical loop. If a system will query for the same archetype every frame, it should create the query object once and reuse it.

## Data Pipeline
This class does not process a stream of data. Instead, it serves as a predicate within a larger data filtering pipeline orchestrated by an entity manager.

> Flow:
> System Logic -> Archetype Definition -> **ExactArchetypeQuery** (Instantiation) -> EntityManager.query(query) -> World Data Structure Traversal -> Filtered Entity Set

