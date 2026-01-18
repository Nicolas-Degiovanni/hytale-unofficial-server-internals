---
description: Architectural reference for BlockSpawner
---

# BlockSpawner

**Package:** com.hypixel.hytale.builtin.blockspawner.state
**Type:** Data Component

## Definition
```java
// Signature
public class BlockSpawner implements Component<ChunkStore> {
```

## Architecture & Concepts
The BlockSpawner class is a state-holding **Data Component** within Hytale's Entity-Component-System (ECS) architecture. It does not contain any logic. Its sole responsibility is to mark a world chunk as a "block spawner" and associate it with a specific spawner definition via an identifier.

This component is attached to a `ChunkStore` entity, effectively binding the spawner's existence and location to a specific chunk in the game world. The `blockSpawnerId` field acts as a foreign key, referencing a detailed spawner configuration likely defined in external JSON or other data files. This configuration would specify parameters such as the types of blocks to spawn, spawn rates, conditions, and patterns.

A separate, logic-driven class, presumably a `BlockSpawnerSystem`, is responsible for querying all `ChunkStore` entities that possess this `BlockSpawner` component. That system then uses the `blockSpawnerId` to fetch the appropriate rules and execute the block spawning logic during the server tick. This separation of data (the component) from behavior (the system) is a core tenet of the engine's design.

## Lifecycle & Ownership
- **Creation:** A BlockSpawner component is typically created and attached to a `ChunkStore` entity during world generation. It can also be added dynamically by server-side game logic, scripts, or administrative commands. The static `CODEC` field indicates it is also instantiated by the persistence layer when a chunk is loaded from disk.
- **Scope:** The component's lifetime is strictly tied to the `ChunkStore` entity it is attached to. It persists as long as the component is not explicitly removed or the parent chunk is unloaded and not saved.
- **Destruction:** The component is destroyed when it is programmatically removed from its parent `ChunkStore` or when the chunk is unloaded from memory without being saved. Garbage collection reclaims the memory.

## Internal State & Concurrency
- **State:** The component's state is mutable and consists of a single `String` field, `blockSpawnerId`. This allows game logic to potentially change the type of spawner active in a chunk at runtime.
- **Thread Safety:** This class is **not thread-safe**. As a simple data container, it possesses no internal locking mechanisms. All access and mutation must be performed on the main server thread to prevent race conditions and data corruption. The engine's ECS framework is responsible for enforcing this single-threaded access during the game tick.

## API Surface
The public API is minimal, focusing exclusively on data management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlockSpawnerId() | String | O(1) | Retrieves the unique identifier for the spawner configuration. |
| setBlockSpawnerId(id) | void | O(1) | Sets or updates the spawner configuration identifier. |
| getComponentType() | ComponentType | O(1) | Static method to retrieve the registered type definition for this component. |

## Integration Patterns

### Standard Usage
The component is intended to be retrieved from a `ChunkStore` entity within a system for processing. Direct manipulation is rare outside of initialization or specific game events.

```java
// Example from within a hypothetical BlockSpawnerSystem
void processSpawners(World world) {
    for (ChunkStore chunk : world.queryEntitiesWith(BlockSpawner.class)) {
        BlockSpawner spawner = chunk.getComponent(BlockSpawner.class);
        String spawnerId = spawner.getBlockSpawnerId();
        
        // Use spawnerId to look up rules and execute spawning logic...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockSpawner()` in general game logic. Components must be managed through their parent entity's component container, for example via `chunkStore.addComponent(new BlockSpawner(...))`.
- **Logic Implementation:** Do not add complex logic, timers, or state machines to this class. It is designed to hold persistent state only. All behavior should be implemented in a dedicated system.
- **Cross-Thread Modification:** Never get a reference to a BlockSpawner component on one thread and modify it from another. All mutations must be synchronized with the main server game loop.

## Data Pipeline
The BlockSpawner component is a key part of the world persistence pipeline.

> **Serialization (World Save):**
> Game Logic modifies `blockSpawnerId` -> `ChunkStore` is marked as dirty -> Server triggers world save -> `ChunkStore` serializes its components -> **BlockSpawner.CODEC** is invoked -> `blockSpawnerId` is written to chunk data on disk.

> **Deserialization (World Load):**
> Server loads a chunk from disk -> `ChunkStore` entity is created -> Chunk data is read -> **BlockSpawner.CODEC** is invoked -> A new `BlockSpawner` instance is created from the `blockSpawnerId` -> The new component is attached to the `ChunkStore`.

