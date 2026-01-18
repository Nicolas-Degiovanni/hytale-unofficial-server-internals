---
description: Architectural reference for SpawnBeaconCheckRemovalSystem
---

# SpawnBeaconCheckRemovalSystem

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.systems
**Type:** Engine Service

## Definition
```java
// Signature
public class SpawnBeaconCheckRemovalSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The SpawnBeaconCheckRemovalSystem is a server-side Entity-Component-System (ECS) service responsible for maintaining data integrity between spawn beacons and their associated objectives. It functions as an automated cleanup and validation mechanism within the game's world state.

Its primary role is to prevent orphaned `LegacySpawnBeaconEntity` components from existing in the world. An orphaned beacon is one that references an objective UUID that no longer corresponds to a valid, loaded objective in the `ObjectiveDataStore`. This scenario can arise from partial data loading, quest state rollbacks, or manual world editing.

By subscribing to the creation of entities with the `LegacySpawnBeaconEntity` component, this system immediately cross-references the entity's objective link. If the link is broken, the system proactively removes the component, which in turn triggers the removal of the entire entity from the world. This self-healing behavior is critical for preventing world state corruption and ensuring that players do not encounter non-functional or broken game elements.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's core ECS System Manager during the server bootstrap sequence. The engine discovers and registers all `HolderSystem` implementations.
- **Scope:** The system is a long-lived service. A single instance persists for the entire duration of a server session.
- **Destruction:** The instance is destroyed and garbage collected when the server shuts down and the ECS System Manager is torn down.

## Internal State & Concurrency
- **State:** This system is **stateless**. It maintains no internal fields, caches, or data between invocations. Its logic is purely reactive and depends only on the arguments provided to its methods by the ECS engine.
- **Thread Safety:** This system is **not thread-safe** for external invocation. However, it is designed to be operated exclusively by the Hytale ECS engine, which guarantees that its methods (`onEntityAdd`, `onEntityRemoved`) are called from a single, predictable thread (typically the main server tick thread). No synchronization primitives are required as concurrent access is not a valid operational model.

## API Surface
The public contract is inherited from `HolderSystem` and invoked by the engine, not by user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns a query for the `LegacySpawnBeaconEntity` component type. This registers the system's interest in entities possessing this component. |
| onEntityAdd(holder, reason, store) | void | O(1) | Callback triggered by the engine when a matching entity is added. Contains the core validation logic. |
| onEntityRemoved(holder, reason, store) | void | O(1) | Callback triggered by the engine when a matching entity is removed. This implementation is a no-op. |

## Integration Patterns

### Standard Usage
This system is not used directly. Its functionality is implicitly activated by the engine. A developer's interaction is limited to creating an entity with a `LegacySpawnBeaconEntity` component. The system will then automatically validate it upon its addition to the world.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SpawnBeaconCheckRemovalSystem()`. The ECS engine is solely responsible for its lifecycle. Manually creating an instance will result in a non-functional object that is not registered to receive engine events.
- **Manual Invocation:** Never call `onEntityAdd` or other lifecycle methods directly. Doing so bypasses the engine's transactional integrity and state management, which can lead to race conditions, corrupted game state, or server crashes.

## Data Pipeline
This system acts as a reactive validator in the entity creation pipeline. It intercepts the addition of specific components to perform a cross-system check.

> Flow:
> Entity Spawning (e.g., from world load) -> `EntityStore` receives new entity with `LegacySpawnBeaconEntity` component -> ECS Engine dispatches "add" event -> **SpawnBeaconCheckRemovalSystem.onEntityAdd** -> System queries `ObjectiveDataStore` -> If objective is missing, system calls `component.remove()` -> ECS Engine processes the component removal, deleting the entity.

