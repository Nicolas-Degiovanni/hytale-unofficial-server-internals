---
description: Architectural reference for ChunkSpawnedNPCData
---

# ChunkSpawnedNPCData

**Package:** com.hypixel.hytale.server.spawning.world.component
**Type:** Transient Data Component

## Definition
```java
// Signature
public class ChunkSpawnedNPCData implements Component<ChunkStore> {
```

## Architecture & Concepts
ChunkSpawnedNPCData is a data-holding **Component** within Hytale's server-side architecture. Its sole purpose is to attach persistent state to a ChunkStore entity, effectively tracking metrics required by the NPC spawning system.

This component acts as a ledger for a single world chunk. It maintains a map of **Environments** to a numeric value representing the "spawn count" or "spawn pressure" within that chunk. For example, it might record that 5 "forest-type" NPCs have spawned in this chunk. This data is fundamental for the SpawningPlugin to make intelligent decisions, such as:

*   Enforcing population density caps for specific biomes.
*   Preventing runaway NPC spawning in a localized area.
*   Modulating spawn rates based on existing populations.

It is a pure data container; it contains no behavioral logic. The SpawningPlugin is the **System** that reads from and writes to this component to implement the actual spawning rules.

### Lifecycle & Ownership
- **Creation:** An instance is created and attached to a ChunkStore under two conditions:
    1.  When a chunk is loaded from disk and its saved data contains this component. The static CODEC is used to deserialize the data into a new instance.
    2.  When the SpawningPlugin first needs to track spawn data for a newly generated or loaded chunk that does not yet have it.

- **Scope:** The lifecycle of a ChunkSpawnedNPCData instance is strictly bound to its parent ChunkStore. It persists in memory only as long as the chunk is loaded on the server.

- **Destruction:** The component is marked for garbage collection when the parent ChunkStore is unloaded from memory. Its state is serialized to disk via the CODEC just before the chunk is unloaded, ensuring the spawn counts are persistent across server restarts.

## Internal State & Concurrency
- **State:** The component's state is mutable. It centers around the private **environmentSpawnCounts** field, an Int2DoubleMap. This map's contents are expected to be modified frequently by the spawning system during gameplay.

- **Thread Safety:** **This component is not thread-safe.** It is designed to be owned and operated by the single main server thread that manages world ticks. Any attempt to read or modify its state from an asynchronous task or worker thread will result in data corruption or a ConcurrentModificationException. All interactions must be scheduled on the main game loop.

## API Surface
The public API is minimal, focusing exclusively on state management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for this component class. |
| getEnvironmentSpawnCount(int) | double | O(1) | Retrieves the current spawn count for a given environment ID. |
| setEnvironmentSpawnCount(int, double) | void | O(1) | Sets or updates the spawn count for a given environment ID. |
| clone() | Component | O(N) | Creates a deep copy of the component and its internal map. N is the number of tracked environments. |

## Integration Patterns

### Standard Usage
Server-side systems should retrieve this component from a ChunkStore instance to query or update spawn data within a single game tick.

```java
// A system running on the server's main thread
// Assume 'chunkStore' is a valid, loaded chunk instance
// Assume 'forestEnvironmentId' is the integer ID for the forest environment

ChunkSpawnedNPCData spawnData = chunkStore.getComponent(ChunkSpawnedNPCData.getComponentType());

if (spawnData != null) {
    double currentForestSpawns = spawnData.getEnvironmentSpawnCount(forestEnvironmentId);

    // Logic to decide if a new spawn is allowed
    if (currentForestSpawns < MAX_FOREST_SPAWNS) {
        // ... spawn an NPC ...
        spawnData.setEnvironmentSpawnCount(forestEnvironmentId, currentForestSpawns + 1);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using **new ChunkSpawnedNPCData()**. The component lifecycle is managed by the ChunkStore and the broader component framework. Incorrectly creating an instance will result in an untracked component that will not be persisted.

- **Cross-Tick Caching:** Do not retrieve a spawn count, store it in a local variable, and use that value in a later game tick. The authoritative state is within the component, and other systems may have modified it. Always query the component for the most up-to-date value when you need it.

- **Asynchronous Modification:** Never call **setEnvironmentSpawnCount** from a separate thread. This is a critical violation of the server's threading model and will lead to unpredictable state corruption.

## Data Pipeline
The component's primary data pipeline involves serialization for world saving and deserialization for world loading. The CODEC handles a crucial transformation during this process.

> **Serialization (World Save)**:
> In-Memory `ChunkSpawnedNPCData` (using integer IDs) → **CODEC** → Translates integer IDs to human-readable string IDs (e.g., 12 -> "hytale:forest") → Serialized Map → Chunk NBT Data → Disk

> **Deserialization (World Load)**:
> Disk → Chunk NBT Data → **CODEC** → Translates string IDs back to optimized integer IDs → New `ChunkSpawnedNPCData` Instance → Attached to `ChunkStore`

