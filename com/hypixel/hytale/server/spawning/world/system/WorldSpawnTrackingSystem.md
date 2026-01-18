---
description: Architectural reference for WorldSpawnTrackingSystem
---

# WorldSpawnTrackingSystem

**Package:** com.hypixel.hytale.server.spawning.world.system
**Type:** System Component

## Definition
```java
// Signature
public class WorldSpawnTrackingSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The WorldSpawnTrackingSystem is a critical component of the server's population control mechanism. It functions as a reactive, event-driven bookkeeper within the Entity Component System (ECS) framework. Its sole responsibility is to maintain an accurate, real-time census of Non-Player Characters (NPCs) across the entire world and within individual chunks.

This system does not decide *when* or *where* to spawn entities. Instead, it listens for entity creation and destruction events and updates the world's population metrics accordingly. These metrics are then consumed by other systems, such as the WorldSpawner, to make informed decisions about spawning new entities, thereby preventing overpopulation and ensuring biome-specific diversity.

The core architectural pattern is the **Separation of Tracking and Spawning**. This system only tracks; other systems act on the data it maintains.

A key concept implemented by this system is **Spawn Pressure Spreading**. When an NPC is added or removed, its impact on the population count is not confined to the single chunk it occupies. The system uses a SpiralIterator to distribute a fractional "spawn cost" to neighboring chunks within a fixed radius (COUNT_SPREAD_RADIUS). This design prevents population counts from becoming overly concentrated in a single chunk, encouraging a more natural and distributed placement of NPCs by the spawning logic.

It operates on three primary data structures:
1.  **WorldSpawnData:** A global ECS Resource that tracks the total expected versus actual NPC counts for each environment type across the entire world.
2.  **ChunkSpawnData:** A per-chunk ECS Component that defines the spawn capacity and rules for that specific chunk.
3.  **ChunkSpawnedNPCData:** A per-chunk ECS Component that stores the current, fractional count of NPCs attributed to that chunk, reflecting the effects of spawn pressure spreading.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's core `SystemManager` during the initialization of a `World`. It is not intended for manual creation.
-   **Scope:** The lifecycle of a WorldSpawnTrackingSystem instance is tightly coupled to its parent `World`. It persists for the entire duration that the world is loaded and active on the server.
-   **Destruction:** The instance is garbage collected when the `World` is unloaded and the `SystemManager` that owns it is shut down.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. It holds immutable references to `ComponentType` and `ResourceType` handles, which are used to access the actual state stored externally within the ECS framework's component stores. All mutable state it modifies (e.g., NPC counts) resides in `WorldSpawnData` and `ChunkSpawnedNPCData` components.
-   **Thread Safety:** **This system is not thread-safe.** It is designed to be operated exclusively by the single main thread that manages the world's tick cycle. Its methods directly manipulate component data without locks, assuming serialized access provided by the parent ECS engine. Invoking its methods from any other thread will lead to data corruption, race conditions, and server instability.

## API Surface
The public API is dictated by its `RefSystem` inheritance and is intended for invocation only by the ECS engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the query that subscribes this system to entities possessing both NPCEntity and TransformComponent. |
| onEntityAdded(ref, reason, store, cmd) | void | O(k) | Callback triggered when a matching NPC entity is added to the world. Increments global and local spawn counts. The complexity is constant based on the fixed spread radius *k*. |
| onEntityRemove(ref, reason, store, cmd) | void | O(k) | Callback triggered when a matching NPC entity is removed from the world. Decrements global and local spawn counts. The complexity is constant based on the fixed spread radius *k*. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Its functionality is invoked implicitly by the engine whenever an entity matching its query is created or destroyed. The standard "usage" is to simply spawn an NPC entity, and trust that this system will automatically track it.

```java
// Example of another system triggering WorldSpawnTrackingSystem implicitly
// This code would exist in a different system, like a quest or spawner system.

// 1. Create an entity with the required components.
Ref<EntityStore> newNpcRef = entityStore.createEntity();
commandBuffer.addComponent(newNpcRef, new NPCEntity(...));
commandBuffer.addComponent(newNpcRef, new TransformComponent(...));

// 2. When the command buffer is processed, the ECS engine will notify
//    WorldSpawnTrackingSystem.onEntityAdded for this new entity.
//    No direct call is ever made.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new WorldSpawnTrackingSystem()`. The system must be registered with and managed by the world's `SystemManager` to function correctly.
-   **Manual Invocation:** Do not call `onEntityAdded` or `onEntityRemove` directly. This bypasses the ECS engine's state management and will corrupt the world's spawn counts. Entity lifecycle events must be the sole trigger for this system's logic.
-   **Spawning Untracked NPCs:** Creating a mobile NPC entity *without* both an NPCEntity and a TransformComponent will cause it to be ignored by this system, leading to an inaccurate population census and potentially uncontrolled spawning.

## Data Pipeline
The system acts as a processor for entity lifecycle events, transforming them into updates for world and chunk-level data components.

> **Flow on NPC Addition:**
> Entity Spawn Event -> ECS Engine Dispatcher -> **WorldSpawnTrackingSystem.onEntityAdded** -> Updates `WorldSpawnData` (Global Count) -> Updates `ChunkSpawnedNPCData` (Origin Chunk) -> Spreads fractional updates to neighboring `ChunkSpawnedNPCData` components via SpiralIterator.

> **Flow on NPC Removal:**
> Entity Despawn Event -> ECS Engine Dispatcher -> **WorldSpawnTrackingSystem.onEntityRemove** -> Updates `WorldSpawnData` (Global Count) -> Updates `ChunkSpawnedNPCData` (Origin Chunk) -> Spreads fractional updates to neighboring `ChunkSpawnedNPCData` components via SpiralIterator.

