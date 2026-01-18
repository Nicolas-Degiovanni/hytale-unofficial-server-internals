---
description: Architectural reference for FloodFillPositionSelector
---

# FloodFillPositionSelector

**Package:** com.hypixel.hytale.server.spawning.util
**Type:** Transient Component

## Definition
```java
// Signature
public class FloodFillPositionSelector implements Component<EntityStore> {
```

## Architecture & Concepts
The FloodFillPositionSelector is a specialized, high-performance component responsible for identifying and caching valid spawn locations for NPCs within a defined radius. It serves as a core analytical tool for the server's spawning subsystem, bridging the gap between static world geometry and dynamic spawning rules.

Its primary function is to solve the computationally expensive problem of finding suitable ground positions in a complex 3D environment. To achieve this efficiently, it employs a two-phase strategy:

1.  **Area Analysis via Flood Fill:** The component first performs a flood fill algorithm starting from a given origin point. This process explores the local terrain, identifying all reachable, non-obstructed ground-level blocks within the specified vertical and horizontal range. The results are stored in a `heightGrid`, which maps each (X, Z) coordinate in the area to a valid spawnable Y-coordinate.

2.  **Multi-Resolution Caching:** To avoid re-evaluating the entire area on every spawn attempt, the component generates a hierarchy of lower-resolution summary maps from the initial `heightGrid`. These maps act as a spatial index, allowing the system to quickly identify promising regions for spawning without iterating over every single block. This is critical for performance, as it reduces a large search space into a small set of candidate zones.

This component is designed to be instantiated for a specific spawning operation, caching its results for subsequent queries until it is invalidated.

### Lifecycle & Ownership
The lifecycle of a FloodFillPositionSelector instance is tightly coupled to a single, continuous spawning evaluation for a specific area, such as a monster spawner or a zone-based spawn beacon.

-   **Creation:** An instance is typically created by a higher-level spawning manager when a spawn beacon becomes active. It is initialized with a reference to the World and a BeaconSpawnWrapper, which provides the configuration for the search (e.g., radius, vertical range). The implementation of the clone method suggests it may also be created via a prototype pattern.

-   **Scope:** The object is stateful and short-lived. Its internal cache is valid only for the duration of a single `buildPositionCache` execution. It is designed to be used, potentially over several game ticks, and then discarded or reset once the spawning conditions change significantly (e.g., terrain modification) or the associated beacon is deactivated.

-   **Destruction:** The object is subject to standard Java garbage collection once all references to it are released. The `buildPositionCache` method explicitly releases its internal SpawningContext, indicating it manages pooled resources that must be returned after the analysis is complete.

## Internal State & Concurrency
-   **State:** The component is highly mutable. Its primary purpose is to build and hold a complex internal state representing the valid spawnable locations in an area. Key state variables include the `heightGrid`, the multi-resolution `resolutionMaps`, and the `positionsByRole` cache. This state is populated by the `buildPositionCache` method and is considered invalid until that method has run to completion.

-   **Thread Safety:** **This component is not thread-safe.** Its internal collections and state variables are accessed and modified without any synchronization mechanisms. Concurrent calls to its methods, especially `buildPositionCache` or `prepareSpawnContext`, from multiple threads will result in race conditions, data corruption, and unpredictable behavior. Each instance must be confined to a single thread. The use of ThreadLocal for its sorting buffer further reinforces this single-threaded design assumption.

## API Surface
The public API provides a contract for initializing, executing, and querying the position selection process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init() | void | O(N) | Resets the internal state, clearing all grids and caches. Must be called before rebuilding the cache. |
| buildPositionCache(origin, pool) | void | O(W*H) | **Core Method.** Executes the flood fill and builds the position cache. This is a computationally expensive, I/O-heavy operation that reads chunk data. W and H are the width and height of the search area. |
| prepareSpawnContext(...) | boolean | O(k log k) | Selects a final, weighted spawn position from the cache based on player proximity. Sorts a subset of *k* candidate positions. Returns false if no suitable position can be found. |
| shouldRebuildCache() | boolean | O(1) | Returns true if the internal cache is considered stale and needs to be rebuilt. |
| forceRebuildCache() | void | O(1) | Manually invalidates the cache, ensuring `shouldRebuildCache` returns true. |

## Integration Patterns

### Standard Usage
The component is intended to be used in a stateful manner by a spawning controller. The expensive analysis is performed once, and the results are used for subsequent spawning attempts.

```java
// Assumes 'selector' is an instance managed by a spawning system
// and 'origin' is the center of the spawn area.

if (selector.shouldRebuildCache()) {
    selector.init();
    selector.buildPositionCache(origin, pool);
}

// On a later tick, when attempting to spawn an NPC
if (selector.hasPositionsForRole(someRoleIndex)) {
    SpawningContext context = new SpawningContext();
    boolean foundPosition = selector.prepareSpawnContext(
        player.getPosition(),
        spawnsThisRound,
        someRoleIndex,
        context,
        spawnWrapper
    );

    if (foundPosition) {
        // Use the populated 'context' to spawn the entity
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Do not access a FloodFillPositionSelector instance from multiple threads. All method calls must be synchronized externally or confined to a single worker thread.
-   **Stateful Reuse without Reset:** Do not call `buildPositionCache` multiple times on the same instance without calling `init` in between. Doing so will result in corrupted and merged cache data from different locations.
-   **Ignoring Cache Invalidation:** Do not indefinitely use a cached selector without checking `shouldRebuildCache` or calling `forceRebuildCache` when the environment changes. This can lead to failed spawn attempts or spawning in invalid locations (e.g., inside a player-built wall).

## Data Pipeline
The component transforms high-level spawn configuration into a set of concrete, validated world coordinates.

> Flow:
> BeaconSpawnWrapper (Config) + Vector3d (Origin) -> **buildPositionCache** -> ChunkAccessor (Reads World Data) -> Flood Fill Algorithm -> **heightGrid** (Surface Map) -> Multi-Resolution BitMaps -> **findPositions** -> `canSpawn` (Validation against Suppression, Light, Blocks) -> **positionsByRole** (Internal Cache) -> `prepareSpawnContext` -> Player Position (Dynamic Weighting) -> **SpawningContext** (Final Output)

