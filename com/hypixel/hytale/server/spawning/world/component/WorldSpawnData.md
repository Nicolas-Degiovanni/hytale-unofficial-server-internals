---
description: Architectural reference for WorldSpawnData
---

# WorldSpawnData

**Package:** com.hypixel.hytale.server.spawning.world.component
**Type:** Component Resource

## Definition
```java
// Signature
public class WorldSpawnData implements Resource<EntityStore> {
```

## Architecture & Concepts
The WorldSpawnData class is the central state aggregator and statistical cache for all NPC spawning activities within a single World instance. It functions as a managed **Resource** attached to a world's EntityStore, effectively making it a world-level component that tracks the complete spawning ecosystem.

Its primary responsibility is to maintain an accurate, real-time census of all spawned NPCs, categorized by their environment. This allows the spawning system to make informed decisions based on population caps, spawn densities, and other configured parameters. It does not decide *which* NPC to spawn, but rather provides the quantitative data needed for other systems to make that decision.

The core of its state is a map from an environment's unique integer ID to a corresponding WorldEnvironmentSpawnData object, which contains the fine-grained statistics for that specific biome or area type. This class aggregates these individual statistics into a world-global view.

Furthermore, it tracks the lifecycle and performance of asynchronous spawn jobs, providing metrics on active jobs, budget usage, and completion rates. This is critical for monitoring and balancing server performance against spawning throughput.

## Lifecycle & Ownership
- **Creation:** WorldSpawnData is never instantiated directly via its constructor. As a Resource, it is created on-demand by the server's resource management system. It is typically instantiated the first time the SpawningPlugin requests it from a World's ComponentAccessor for a newly loaded world.

- **Scope:** The lifecycle of a WorldSpawnData instance is strictly bound to the lifecycle of the World it represents. It persists in memory for the entire duration that the world is loaded on the server.

- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when its parent World is unloaded from the server. It has no explicit destruction or cleanup method. The `clone` method is explicitly unsupported, reinforcing that this component represents the unique, live state of a specific world.

## Internal State & Concurrency
- **State:** This component is highly **mutable**. It is designed to be a live, in-memory representation of the world's spawning state, which changes continuously as NPCs spawn and despawn. It caches per-environment data to avoid redundant calculations.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. It uses non-concurrent collections like Int2ObjectOpenHashMap and ArrayDeque for performance. All method calls that mutate state must be synchronized externally. In practice, this means all interactions with a WorldSpawnData instance must occur on the main server thread (the world tick thread). Data from asynchronous spawn jobs must be marshaled back to the main thread before being used to update this component.

## API Surface
The public API provides methods for tracking NPC populations, managing spawn job metrics, and querying the aggregated world state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOrCreateWorldEnvironmentSpawnData(envIndex, world, accessor) | WorldEnvironmentSpawnData | O(N) | Lazily creates and initializes the data tracker for a specific environment. Complexity is tied to the number of spawn configurations (N) for that environment. |
| trackNPC(envIndex, roleIndex, count, world, accessor) | boolean | O(1) | Increments the count for a specific NPC role within an environment. This is a primary entry point for state mutation. |
| untrackNPC(envIndex, roleIndex, count) | boolean | O(1) | Decrements the count for a specific NPC role within an environment. |
| recalculateWorldCount() | void | O(E) | Recalculates world-total expected and actual NPC counts by iterating over all known environments (E). Essential for maintaining state consistency. |
| updateSpawnability() | void | O(E) | Updates the world-level `unspawnable` flag by checking the state of all tracked environments. |
| adjustActiveSpawnJobs(amount, trackedCount) | void | O(1) | Updates counters related to asynchronous spawn jobs. Must be called from the main server thread. |
| queueUnspawnableChunk(envIndex, chunkIndex) | void | O(1) | Adds a chunk that failed to produce a spawn to a processing queue for later analysis. |
| nextUnspawnableChunk() | UnspawnableEntry | O(1) | Retrieves the next unspawnable chunk from the internal queue for processing. |

## Integration Patterns

### Standard Usage
The WorldSpawnData component should always be retrieved from the target World's ComponentAccessor. It is used by the core spawning system to check population caps and determine if new spawn attempts are necessary.

```java
// Standard retrieval and use within the spawning system
ComponentAccessor<EntityStore> accessor = world.getComponentAccessor();
WorldSpawnData spawnData = accessor.getResource(WorldSpawnData.getResourceType());

// Recalculate totals to ensure data is fresh before making decisions
spawnData.recalculateWorldCount();

// Check if the world's NPC population is below the desired threshold
if (spawnData.getActualNPCs() < spawnData.getExpectedNPCs()) {
    // Logic to initiate new spawn jobs...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldSpawnData()`. This creates an unmanaged object that is not tied to any world's lifecycle, leading to memory leaks and incorrect behavior.
- **Multi-threaded Mutation:** Do not call methods like `trackNPC` or `adjustActiveSpawnJobs` from worker threads. This will cause race conditions and corrupt the world's spawning state. All mutations must be synchronized on the main world thread.
- **State Staleness:** Failing to call `recalculateWorldCount` periodically or after significant state changes (e.g., a mass despawn event) can cause the spawning system to operate on stale data, leading to over-spawning or under-spawning.

## Data Pipeline
WorldSpawnData acts as a central sink for events related to NPC population changes and as a source of truth for spawning decision logic.

> **Flow 1: NPC Spawn Event**
> NPC Entity Created -> Spawning System -> **WorldSpawnData.trackNPC()** -> Internal counters and maps are updated.

> **Flow 2: Spawning System Tick**
> World Tick -> Spawning System Logic -> **WorldSpawnData.getActualNPCs()** & **WorldSpawnData.getExpectedNPCs()** -> Decision to spawn is made -> Spawn Job is dispatched.

> **Flow 3: Spawn Job Completion**
> Async Spawn Job Finishes -> Result posted to Main Thread -> **WorldSpawnData.addCompletedSpawnJob()** -> Performance metrics are updated.

