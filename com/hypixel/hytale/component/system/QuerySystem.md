---
description: Architectural reference for QuerySystem
---

# QuerySystem

**Package:** com.hypixel.hytale.component.system
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface QuerySystem<ECS_TYPE> extends ISystem<ECS_TYPE> {
```

## Architecture & Concepts
The **QuerySystem** interface defines a specialized contract within Hytale's Entity-Component-System (ECS) architecture. It extends the base **ISystem** interface, adding a powerful filtering mechanism for performance and architectural clarity.

At its core, a **QuerySystem** is a system that declares it only operates on a specific subset of entities. This subset is defined by a **Query** object, which specifies a required signature of components. The engine's **SystemManager** uses this query to identify matching **Archetypes**—groups of entities that all share the same component structure.

This design pattern is a critical optimization. Instead of iterating over every entity in the world during each game tick, a **QuerySystem** implementation is only invoked for the archetypes that satisfy its query. This dramatically reduces computational overhead, especially in worlds with a large number and diversity of entities. It serves as the primary mechanism for associating system logic with specific entity types in a decoupled manner.

## Lifecycle & Ownership
As an interface, **QuerySystem** itself has no lifecycle. The following applies to concrete classes that *implement* this interface.

- **Creation:** Implementations are typically discovered via reflection or explicit registration and instantiated by a central **SystemManager** or **World** context during the world-loading phase.
- **Scope:** The lifecycle of a **QuerySystem** implementation is tied to the game world it operates on. It persists as long as the world is active.
- **Destruction:** Instances are marked for garbage collection when the associated game world is unloaded or the server/client session terminates.

## Internal State & Concurrency
The **QuerySystem** contract itself is stateless. However, implementations are expected to be stateful, often containing logic and data relevant to the entities they process.

- **State:** The interface methods are pure contracts for retrieving a filter. Concrete implementations may contain mutable state, but this is strongly discouraged for any data not directly related to the system's immediate execution context.
- **Thread Safety:** **WARNING:** Implementations of **QuerySystem** are not guaranteed to be thread-safe. The ECS framework is designed to execute systems in a deterministic, single-threaded sequence within the main game loop. All state mutations should occur exclusively within the processing methods (e.g., `update`) to prevent race conditions and ensure world-state consistency. Do not access or modify a system's state from asynchronous tasks without explicit synchronization, which is considered an anti-pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(componentRegistry, archetype) | boolean | O(C) | Default method. Tests if a given **Archetype** matches the system's **Query**. Complexity is proportional to the number of components (C) in the query. |
| getQuery() | Query | O(1) | **CRITICAL:** Returns the **Query** object that defines which entities this system operates on. Must not be null. |

## Integration Patterns

### Standard Usage
Developers do not call methods on **QuerySystem** directly. Instead, they implement the interface to create a new system that will be automatically discovered and managed by the engine. The primary responsibility is to provide a stable **Query** object.

```java
// A system that processes all entities with both a Position and Velocity component.
public class MovementSystem implements QuerySystem<ServerWorld> {

    // The query is typically a static final field for performance.
    private static final Query<ServerWorld> QUERY = Query.Builder
        .all(Position.class, Velocity.class)
        .build();

    @Override
    public Query<ServerWorld> getQuery() {
        return QUERY;
    }

    @Override
    public void update(ECSContext<ServerWorld> context, float deltaTime) {
        // The engine invokes this method only for entities matching the query.
        // ... implementation to update entity positions based on velocity ...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Dynamic Queries:** Do not create a new **Query** instance inside `getQuery`. This is highly inefficient and will cause significant performance degradation, as the engine may re-evaluate archetype matches unnecessarily. Queries should be static and immutable.
- **Null Query:** Never return null from `getQuery`. This violates the contract and will result in a runtime exception or undefined behavior within the **SystemManager**.
- **Bypassing the Query:** Do not implement **QuerySystem** and then proceed to iterate over all world entities inside the `update` method. This completely defeats the purpose and performance benefits of the query mechanism.

## Data Pipeline
The **QuerySystem** acts as a filter in the engine's main processing pipeline. It does not transform data itself but rather gates the flow of entity collections to system logic.

> Control Flow:
> Game Loop Tick -> SystemManager.updateAll() -> For each Archetype -> **QuerySystem.test(archetype)** -> (If match) -> System.update(matchingEntities) -> World State Mutation

