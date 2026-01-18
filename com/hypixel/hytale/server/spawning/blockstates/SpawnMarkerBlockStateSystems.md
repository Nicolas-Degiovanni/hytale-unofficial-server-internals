---
description: Architectural reference for SpawnMarkerBlockStateSystems
---

# SpawnMarkerBlockStateSystems

**Package:** com.hypixel.hytale.server.spawning.blockstates
**Type:** Utility / System Container

## Definition
```java
// This class is a container for related ECS systems.
// It is not intended to be instantiated.
public class SpawnMarkerBlockStateSystems {
    // Contains four public static nested System classes.
}
```

## Architecture & Concepts
The SpawnMarkerBlockStateSystems class is a crucial component of the server's spawning architecture, acting as a robust bridge between the world's block data and the entity system. Its primary responsibility is to manage a persistent, two-way link between a special block state, the SpawnMarkerBlockState, and a corresponding world entity, the SpawnMarkerEntity.

This class is not a single system but a logical grouping of four distinct Entity Component Systems (ECS) that work in concert to create a self-healing and resilient relationship. The core design principle is that a specific block in the world *owns* and is the single source of truth for a corresponding spawn marker entity. If the block is destroyed, the entity must be removed. If the entity is removed for any reason, the block must recreate it.

This two-way binding ensures that spawn markers, which are critical for game logic, cannot easily be desynchronized from the world state. The systems operate across two different data stores: the ChunkStore, which manages block and block-entity data, and the EntityStore, which manages all world entities.

The four internal systems each have a specialized role:

*   **TickHeartbeat:** Operates on SpawnMarkerBlockState components in the ChunkStore. Its job is to periodically check if its corresponding SpawnMarkerEntity exists. If the entity is missing, this system is responsible for creating it. This is the "creator" system.
*   **AddOrRemove:** Also operates on SpawnMarkerBlockState components. It listens for when these components are removed (e.g., the block is broken). When a removal occurs, it schedules the deletion of the associated SpawnMarkerEntity. This is the "destroyer" system.
*   **SpawnMarkerTickHeartbeat:** Operates on SpawnMarkerEntity components in the EntityStore. It performs the reverse check of TickHeartbeat, ensuring the source block that created it still exists and is of the correct type. If the source block is missing or has changed, this system removes the entity to prevent orphans. This is the "guardian" system.
*   **SpawnMarkerAddedFromExternal:** A safety system that operates on SpawnMarkerEntity components. It immediately removes any spawn marker entities that are added to the world via world generation or prefabs. This enforces the strict architectural rule that spawn markers can *only* be created by a SpawnMarkerBlockState. This is the "gatekeeper" system.

### Lifecycle & Ownership
-   **Creation:** The nested system classes within SpawnMarkerBlockStateSystems are instantiated by the core server engine during the world's initialization phase. They are registered with the ECS scheduler to participate in the game loop.
-   **Scope:** These systems are singletons within the context of a World instance. They persist for the entire lifetime of the world.
-   **Destruction:** The systems are destroyed and cleaned up when the World instance is unloaded or the server shuts down. Developers should never create or destroy these systems manually.

## Internal State & Concurrency
-   **State:** The SpawnMarkerBlockStateSystems class itself is stateless. The nested system classes are also effectively stateless, holding only configuration data like the ComponentType they query for. All mutable state is stored within the components they operate on, such as SpawnMarkerBlockState and SpawnMarkerBlockReference.

-   **Thread Safety:** **Not thread-safe.** The ticking systems explicitly declare `isParallel` as false. This is a critical design choice to prevent severe race conditions. Operations often require reading from one store (e.g., ChunkStore) and writing to another (e.g., EntityStore). Such cross-store operations are not atomic. By forcing these systems to run serially, the engine guarantees that the state of the world remains consistent during their execution. The AddOrRemove system further ensures safety by scheduling entity removal on the main world thread via `world.execute`, deferring the action to a safe execution point.

## API Surface
This class and its nested systems do not expose a public API for general developer use. Interaction with this system is entirely indirect, through the placement and removal of blocks that use the SpawnMarkerBlockState. The systems' public methods are contracts for the ECS scheduler, not for game logic.

## Integration Patterns

### Standard Usage
A level designer or procedural world generator interacts with this system by placing a block configured with a SpawnMarkerBlockState. The systems will then automatically handle the creation and lifecycle management of the corresponding SpawnMarkerEntity.

```java
// PSEUDOCODE: How the engine uses this system indirectly

// 1. A block is placed in the world.
WorldChunk chunk = world.getChunkAt(x, z);
Block newBlock = BlockRegistry.get("my_spawn_marker_block");
chunk.setBlock(x, y, z, newBlock);

// 2. The block's type automatically adds a SpawnMarkerBlockState component
//    to the block entity in the ChunkStore.

// 3. On a subsequent server tick, the TickHeartbeat system runs.
//    It finds the new SpawnMarkerBlockState without a linked entity.
//    It calls the internal createMarker method, spawning a SpawnMarkerEntity
//    in the EntityStore and linking it back to the block state.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new` to create instances of the nested system classes. They are managed exclusively by the engine's ECS framework.
-   **Manual Entity Creation:** Do not create a SpawnMarkerEntity and add it to the world directly. The SpawnMarkerAddedFromExternal system will immediately detect and delete it. The only valid way to create a spawn marker is by placing the corresponding block.
-   **Cross-Thread Access:** Do not attempt to call any methods on these systems from other threads. They are designed to be driven only by the main server tick loop.

## Data Pipeline
The flow of data and control is a continuous, self-healing loop between the ChunkStore and the EntityStore, mediated by these systems.

> **Creation Flow:**
> World Gen / Player Action → Block with SpawnMarkerBlockState placed in Chunk → **TickHeartbeat** system detects unlinked state → `createMarker` is called → SpawnMarkerEntity created in EntityStore with reference back to the block.

> **Destruction Flow:**
> Player Action → Block is destroyed → SpawnMarkerBlockState is removed from ChunkStore → **AddOrRemove** system detects removal → Schedules removal of the corresponding SpawnMarkerEntity from the EntityStore.

> **Entity Desync / Healing Flow:**
> Admin Command / Bug → SpawnMarkerEntity is deleted → **TickHeartbeat** system detects the broken link on the SpawnMarkerBlockState → A new SpawnMarkerEntity is created to replace the old one.

> **Block Desync / Healing Flow:**
> Chunk Unloads / Corruption → SpawnMarkerBlockState is no longer accessible → **SpawnMarkerTickHeartbeat** system on the entity fails to find its source block → After a timeout, the entity removes itself to prevent orphans.

