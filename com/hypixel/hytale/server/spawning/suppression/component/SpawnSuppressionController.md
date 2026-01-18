---
description: Architectural reference for SpawnSuppressionController
---

# SpawnSuppressionController

**Package:** com.hypixel.hytale.server.spawning.suppression.component
**Type:** Component (Resource)

## Definition
```java
// Signature
public class SpawnSuppressionController implements Resource<EntityStore> {
```

## Architecture & Concepts

The SpawnSuppressionController is a server-side data component that manages regions where entity spawning is actively prevented. It acts as the central authority for the spawning system to query whether a given area is suppressed. This component is attached to an EntityStore, effectively making it a world-level controller for a specific dimension or world instance.

Its architecture is built around a two-level caching and persistence strategy:

1.  **Persistent State (spawnSuppressorMap):** A map of UUIDs to SpawnSuppressorEntry objects. This is the source of truth, containing detailed information about each individual suppressor (e.g., a specific block or entity). This map is serialized to disk as part of the world save data, ensuring suppressors persist across server restarts. The UUID key links a suppressor to its source entity.

2.  **Transient Cache (chunkSuppressionMap):** A highly optimized, in-memory map that uses a packed chunk coordinate (a long) as its key. This map is a derivative of the persistent state and is used for high-performance lookups during the spawning process. By caching suppression status on a per-chunk basis, the spawning algorithm can perform a single, fast check to disqualify an entire chunk from spawning calculations, avoiding costly iterations. This cache is rebuilt when the controller is loaded and updated dynamically.

This component is a critical dependency of the SpawningPlugin, which orchestrates the entire server-side entity spawning lifecycle.

## Lifecycle & Ownership

-   **Creation:** The SpawnSuppressionController is not instantiated directly. It is managed by the engine's resource system. An instance is created and attached to an EntityStore when a world is first generated. On subsequent loads, its state is deserialized from world data via its static CODEC definition.
-   **Scope:** The lifecycle of a SpawnSuppressionController is strictly bound to its parent EntityStore. It persists in memory for the entire duration that the world is loaded on the server.
-   **Destruction:** The component is marked for garbage collection when its parent EntityStore is unloaded, for example, during a server shutdown or when a dimension is no longer active. Prior to destruction, its persistent state (the spawnSuppressorMap) is serialized to disk.

## Internal State & Concurrency

-   **State:** The component's state is mutable. Both internal maps are designed for runtime modification as suppressors are added, removed, or change their area of effect. The spawnSuppressorMap represents the persistent state, while the chunkSuppressionMap is a volatile, in-memory cache.

-   **Thread Safety:** This class is thread-safe. It exclusively uses concurrent map implementations (Long2ObjectConcurrentHashMap and ConcurrentHashMap). This is a mandatory design requirement, as the server's spawning system may operate on multiple worker threads to evaluate different regions of the world in parallel. All public accessors are safe to call from any thread without external locking.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | static ResourceType | O(1) | Retrieves the unique type identifier used to register and query this component from a resource holder. |
| getSpawnSuppressorMap() | Map | O(1) | Returns the underlying map of persistent suppressor data, keyed by suppressor UUID. |
| getChunkSuppressionMap() | Long2ObjectConcurrentHashMap | O(1) | Returns the high-performance, transient cache of suppressed chunks, keyed by packed chunk coordinates. |
| clone() | Resource | O(N+M) | Creates a new instance and performs a shallow copy of the internal maps. Used by the resource system for snapshotting. |

## Integration Patterns

### Standard Usage

The component should always be retrieved from its parent EntityStore. The primary interaction is to query the chunk suppression map for fast checks.

```java
// Obtain the controller from the world's primary entity store
EntityStore worldEntityStore = world.getEntityStore();
SpawnSuppressionController controller = worldEntityStore.getResource(SpawnSuppressionController.getResourceType());

if (controller == null) {
    // This world does not have spawn suppression enabled; proceed normally.
    return;
}

// During the spawning process for a given chunk
long chunkKey = ChunkUtil.indexChunk(chunkX, chunkZ);
if (controller.getChunkSuppressionMap().containsKey(chunkKey)) {
    // CRITICAL: Abort all spawning attempts for this chunk immediately.
    return;
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new SpawnSuppressionController()`. The engine's resource and persistence systems manage its lifecycle. Direct instantiation will result in a disconnected, non-persistent component that the spawning system will not recognize.
-   **State Inconsistency:** Do not modify the spawnSuppressorMap without also updating the chunkSuppressionMap. Doing so will lead to a desynchronized state where the fast cache check returns incorrect results, potentially allowing spawning in a suppressed area until the cache is next rebuilt. All modifications must be handled by a higher-level system that guarantees atomicity between the two maps.
-   **Assuming Cache Persistence:** Do not write logic that assumes the contents of getChunkSuppressionMap will persist across server restarts. It is a transient cache and will be empty upon initial load before being repopulated.

## Data Pipeline

The SpawnSuppressionController primarily serves as a state repository rather than a data processor. Its two main data flows are for querying and state persistence.

**Query Flow (High Frequency):**
> Flow:
> Spawning System (evaluating chunk at X, Z) → **SpawnSuppressionController**.getChunkSuppressionMap() → Cache Lookup (using packed long key) → Suppression Status (true/false) → Spawning Algorithm Aborts or Proceeds

**Persistence Flow (Low Frequency):**
> Flow:
> World Save Trigger → EntityStore Serialization → **SpawnSuppressionController**.CODEC → Serializes spawnSuppressorMap → World Data on Disk

