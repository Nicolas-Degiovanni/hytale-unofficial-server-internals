---
description: Architectural reference for ArchetypeTickingSystem
---

# ArchetypeTickingSystem

**Package:** com.hypixel.hytale.component.system.tick
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class ArchetypeTickingSystem<ECS_TYPE> extends TickingSystem<ECS_TYPE> implements QuerySystem<ECS_TYPE> {
```

## Architecture & Concepts

The ArchetypeTickingSystem is a foundational abstract class within the Entity Component System (ECS) framework. It represents a piece of game logic that executes once per frame (a "tick") on a specific subset of entities. This class is the primary mechanism for implementing behavior in a data-oriented manner.

It acts as the bridge between the high-level TickingScheduler and the low-level component data stored in ArchetypeChunks. By extending TickingSystem, it signals its participation in the main game loop. By implementing QuerySystem, it defines precisely which entities it operates on, based on their component composition (their Archetype).

The core architectural principle is the separation of data (Components) from behavior (Systems). This class enforces that principle by providing a framework where logic is executed over contiguous blocks of component data. This design is critical for performance, as it maximizes CPU cache efficiency by ensuring that the data required by the system's logic is laid out sequentially in memory.

## Lifecycle & Ownership

-   **Creation:** Concrete implementations of ArchetypeTickingSystem are not created manually. They are discovered and instantiated by the component framework's service loader during the engine's bootstrap sequence. Once created, they are registered with a central scheduler.
-   **Scope:** An instance of a concrete system is a singleton that persists for the entire duration of a game session (e.g., for the lifetime of the client or server process).
-   **Destruction:** The system instance is decommissioned and eligible for garbage collection during the engine's shutdown procedure.

## Internal State & Concurrency

-   **State:** This base class is stateless. Concrete implementations **must** remain stateless. All data related to game state should be stored in Components attached to entities, not in fields within the system class. Storing per-entity state in a system is a severe anti-pattern that breaks parallelization and data-oriented design.
-   **Thread Safety:** This class is not inherently thread-safe. Concurrency is managed by the upstream scheduler (the Store). The ECS scheduler may execute different systems on different threads in parallel. The CommandBuffer parameter in the abstract tick method is the designated mechanism for safely queueing structural changes (e.g., adding/removing entities or components) to be resolved at a later synchronization point, preventing race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(registry, archetype) | boolean | O(C) | Part of the QuerySystem contract. Determines if this system should process entities of the given Archetype. C is the number of components in the query. |
| tick(dt, systemIndex, store) | void | O(N) | The primary entry point called by the scheduler. It delegates to the Store, which iterates over all matching ArchetypeChunks. N is the total number of entities processed. |
| tick(dt, chunk, store, commands) | void | O(M) | **Abstract.** The core logic implementation. Called by the Store for each relevant ArchetypeChunk. M is the number of entities in the chunk. |
| isExplicitQuery() | boolean | O(1) | A hook for subclasses to modify query behavior. Defaults to false. |

## Integration Patterns

### Standard Usage

A developer's primary interaction is to extend this class, implement the abstract tick method with game logic, and provide a component query. The engine handles the rest.

```java
// A concrete system that makes entities with a "Spin" component rotate.
public class SpinSystem extends ArchetypeTickingSystem<ClientECS> {

    // The developer defines the query for which entities this system targets.
    @Override
    public Query<ClientECS> getQuery() {
        return Query.all(Rotation.class, Spin.class);
    }

    // The developer implements the core logic here.
    // This method will be called by the engine for each chunk of matching entities.
    @Override
    public void tick(float dt, @Nonnull ArchetypeChunk<ClientECS> chunk, @Nonnull Store<ClientECS> store, @Nonnull CommandBuffer<ClientECS> commands) {
        ComponentSlice<Rotation> rotations = chunk.getSlice(Rotation.class);
        ComponentSlice<Spin> spins = chunk.getSlice(Spin.class);

        for (int i = 0; i < chunk.size(); i++) {
            Rotation rot = rotations.get(i);
            Spin spin = spins.get(i);
            rot.angle += spin.speed * dt;
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new MySystem()`. Systems must be managed by the engine's service registry to be correctly scheduled and integrated into the game loop.
-   **Storing State:** Do not add fields to your system class to track the state of a specific entity. This violates the ECS paradigm and will cause severe bugs, especially in a multithreaded environment.
-   **Direct Structural Modification:** Do not attempt to add or remove components directly from the Store or an ArchetypeChunk within the tick method. All such operations must be queued via the provided CommandBuffer to ensure thread safety and prevent iterator invalidation.

## Data Pipeline

The flow of execution for a single frame demonstrates how this class fits into the wider engine architecture.

> Flow:
> TickingScheduler -> **ArchetypeTickingSystem.tick(dt, store)** -> Store.tick(system, dt) -> *[Iteration over matching ArchetypeChunks]* -> **ArchetypeTickingSystem.tick(dt, chunk, ...)** -> CommandBuffer

