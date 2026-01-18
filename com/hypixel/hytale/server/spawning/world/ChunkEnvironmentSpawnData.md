---
description: Architectural reference for ChunkEnvironmentSpawnData
---

# ChunkEnvironmentSpawnData

**Package:** com.hypixel.hytale.server.spawning.world
**Type:** Transient

## Definition
```java
// Signature
public class ChunkEnvironmentSpawnData {
```

## Architecture & Concepts
The ChunkEnvironmentSpawnData class is a stateful data container that represents the spawning potential for a *single environment type* within a *single WorldChunk*. It is a fundamental component of the server's NPC spawning system, acting as a transient workspace for calculating and tracking spawn metrics.

For any given chunk, the spawning system may create multiple instances of this class—one for each distinct environment (e.g., Forest, Cave, River) present within that chunk's boundaries. The class encapsulates all the necessary state to answer critical questions for the spawning scheduler:
- How many spawnable locations for this environment exist in this chunk?
- What is the target number of NPCs (the "expected" population) for this environment in this chunk?
- Based on the current population, what is the "spawning pressure" or weight for this location?
- Which specific NPC roles have already failed to spawn here, to prevent wasteful repeated attempts?

This object serves as the bridge between static world analysis (identifying spawnable segments) and the dynamic spawning loop (deciding where and what to spawn).

### Lifecycle & Ownership
- **Creation:** Instances are created on-demand by the server's core spawning manager during the chunk analysis phase. A new instance is typically allocated when a chunk is first evaluated for its spawning properties.
- **Scope:** The lifetime of a ChunkEnvironmentSpawnData object is tightly coupled to the `WorldChunk` it describes. It persists as long as the chunk's spawning metadata is considered valid, but it is not a global or session-scoped object. It is effectively a piece of cached metadata for a chunk.
- **Destruction:** The object is eligible for garbage collection when the parent chunk is unloaded from memory or when the spawning system explicitly invalidates its cache for that chunk.

## Internal State & Concurrency
- **State:** This class is highly **mutable**. Its primary function is to accumulate state during an initial analysis phase (via `registerSegment` and `updateDensity`) and then be further modified during the active spawning process (via `markRoleAsUnspawnable`). It caches the list of possible roles, the calculated target NPC count, and a blacklist of unspawnable roles.

- **Thread Safety:** This class is **not thread-safe**. It uses non-concurrent collections like IntOpenHashSet and performs no internal locking. All interactions with a given instance must be synchronized externally. It is designed to be owned and operated on by a single thread responsible for a specific chunk's logic at any given time.

    **WARNING:** Concurrent modification from multiple threads will lead to data corruption, race conditions, and unpredictable spawning behavior. Access must be strictly controlled by the owning system, typically the main server thread or a dedicated world-region thread.

## API Surface
The public API is designed for a sequential, stateful workflow: initialization, data population, and then querying during the spawn loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(environmentIndex, chunk) | void | O(N) | Initializes state for a given environment. Fetches all possible NPC roles. Must be called first. |
| registerSegment(x, z) | void | O(1) | Increments the count of valid spawnable columns within the chunk for this environment. |
| updateDensity(density) | void | O(1) | Calculates and stores the final `expectedNPCs` based on the total segment count. |
| getWeight(spawnedNPCs) | double | O(1) | Returns the "spawning pressure". A higher value indicates a higher priority for spawning. |
| isFullyPopulated(spawnedNPCs) | boolean | O(1) | Returns true if the current NPC count meets or exceeds the expected count. |
| markRoleAsUnspawnable(roleIndex) | void | O(1) avg | Blacklists a specific NPC role, preventing further spawn attempts for it in this context. |
| isRoleSpawnable(roleIndex) | boolean | O(1) avg | Checks if a given NPC role has been blacklisted. |
| allRolesUnspawnable() | boolean | O(1) | Returns true if every possible role for this environment has been marked as unspawnable. |

## Integration Patterns

### Standard Usage
The class is intended to be used in a multi-stage process orchestrated by a higher-level spawning manager.

```java
// 1. During chunk analysis, an instance is created and populated.
ChunkEnvironmentSpawnData spawnData = new ChunkEnvironmentSpawnData();
spawnData.init(forestEnvironmentId, currentChunk);

// Iterate through the chunk's columns to find valid segments
for (int z = 0; z < 16; z++) {
    for (int x = 0; x < 16; x++) {
        if (isSpawnableForestSegment(x, z)) {
            spawnData.registerSegment(x, z);
        }
    }
}
spawnData.updateDensity(forestDensity);

// 2. During a server tick, the spawning system uses the data.
double currentForestNpcs = countNpcsInChunk(currentChunk, forestEnvironmentId);
if (!spawnData.isFullyPopulated(currentForestNpcs)) {
    double weight = spawnData.getWeight(currentForestNpcs);
    // ...add this chunk to a priority queue for potential spawns...
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse an instance for a different chunk or environment without calling `init` again. The internal state, such as `segmentCount` and `unspawnableRoles`, will be invalid.
- **Premature Querying:** Do not call `getWeight` or `updateDensity` before the `registerSegment` loop is complete. The calculations will be based on incomplete data, leading to incorrect NPC population levels.
- **Concurrent Access:** Do not share a single instance across multiple threads that may modify it. For example, do not have one thread analyzing segments while another attempts to read the spawn weight.

## Data Pipeline
This class acts as a stateful node in the broader NPC spawning data pipeline. It transforms raw world geometry into actionable spawning metrics.

> Flow:
> World Chunk Analysis -> **ChunkEnvironmentSpawnData** (Segment Registration & Density Calculation) -> Spawning Scheduler (Weight Evaluation) -> Spawn Attempt Logic -> **ChunkEnvironmentSpawnData** (Role Blacklisting) -> NPC Entity Creation

