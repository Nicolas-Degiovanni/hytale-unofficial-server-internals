---
description: Architectural reference for ChunkSpawnData
---

# ChunkSpawnData

**Package:** com.hypixel.hytale.server.spawning.world.component
**Type:** Transient

## Definition
```java
// Signature
public class ChunkSpawnData implements Component<ChunkStore> {
```

## Architecture & Concepts
The ChunkSpawnData class is a server-side data component that attaches state to a ChunkStore object. It is a fundamental building block of the server's **Spawning System**, acting as the primary state machine for all spawning activities within a single world chunk.

This component does not contain any spawning logic itself. Instead, it serves as a data container that higher-level systems, such as a SpawningSystem or a dedicated SpawningJob, query and mutate. Its core responsibility is to track chunk-specific spawning cooldowns and environment-specific spawn data, enabling the engine to make efficient, stateful decisions about when and where to spawn entities.

By implementing Component<ChunkStore>, it integrates into a specialized component system designed for world data, distinct from the primary Entity Component System. This allows world chunks to be augmented with complex behaviors like spawning without polluting the core ChunkStore definition.

### Lifecycle & Ownership
- **Creation:** An instance of ChunkSpawnData is never created directly via its constructor. It is attached to a ChunkStore by the SpawningPlugin or a related management system when a chunk becomes a candidate for spawning activity, typically when it is loaded or a player enters its vicinity.
- **Scope:** The lifecycle of a ChunkSpawnData instance is strictly bound to the ChunkStore it is attached to. It persists in memory as long as the parent chunk is loaded and actively managed by the server.
- **Destruction:** The component is marked for garbage collection when its parent ChunkStore is unloaded from memory. There is no explicit destruction method; its removal is managed by the component system.

## Internal State & Concurrency
- **State:** This component is highly **mutable**. Its fields, particularly lastSpawn and the internal chunkEnvironmentSpawnDataMap, are expected to be frequently updated by the spawning system to reflect the current state of the chunk.
- **Thread Safety:** **WARNING:** This component is **not thread-safe**. The underlying map, Int2ObjectOpenHashMap, is unsynchronized, and no internal locking mechanisms are employed. All reads and writes to a ChunkSpawnData instance must be externally synchronized or confined to a single, designated thread, such as the main server world tick thread for that chunk's region. Unmanaged concurrent access will lead to state corruption and severe runtime exceptions.

## API Surface
The public API provides direct access to the spawning state for a given chunk.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEnvironmentSpawnData(int) | ChunkEnvironmentSpawnData | O(1) | Retrieves environment-specific data. Throws NullPointerException if data for the given environment ID does not exist. |
| isOnSpawnCooldown() | boolean | O(1) | Checks if a spawn has occurred recently. Returns true if lastSpawn is not zero. |
| setLastSpawn(long) | void | O(1) | Updates the timestamp of the last successful spawn event, effectively starting a cooldown period. |
| setStarted(boolean) | void | O(1) | A flag to control or indicate the initialization state of spawning within the chunk. |
| clone() | Component | N/A | **Unsupported.** Throws UnsupportedOperationException. This component's state is unique to its ChunkStore and must not be copied. |

## Integration Patterns

### Standard Usage
A managing system retrieves this component from a ChunkStore to evaluate and update the chunk's spawning state. This is the only correct way to interact with ChunkSpawnData.

```java
// A SpawningSystem would execute logic like this
ChunkStore targetChunk = world.getChunkStoreAt(chunkPos);
ChunkSpawnData spawnData = targetChunk.getComponent(ChunkSpawnData.getComponentType());

if (spawnData != null && !spawnData.isOnSpawnCooldown()) {
    // Logic to perform a spawn...
    
    // On successful spawn, update the state
    long currentTime = world.getTime();
    spawnData.setLastSpawn(currentTime);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ChunkSpawnData()`. The component must be managed by the engine's component system to ensure correct lifecycle and attachment to a ChunkStore.
- **Concurrent Modification:** Do not access a ChunkSpawnData instance from multiple threads without an external locking strategy. This will cause race conditions and corrupt the spawning state.
- **State Assumption:** Do not call `getEnvironmentSpawnData(id)` without first ensuring that data for that environment ID has been populated. The method's contract is to throw a NullPointerException for missing data.

## Data Pipeline
ChunkSpawnData does not process data in a pipeline. Instead, it acts as a state repository that is read from and written to by the main Spawning System.

> State Flow:
> Spawning System -> **Reads** `isOnSpawnCooldown` from **ChunkSpawnData** -> Makes spawn decision -> Spawning System -> **Writes** `lastSpawn` timestamp to **ChunkSpawnData**

