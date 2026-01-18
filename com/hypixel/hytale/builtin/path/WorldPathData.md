---
description: Architectural reference for WorldPathData
---

# WorldPathData

**Package:** com.hypixel.hytale.builtin.path
**Type:** Component / Data Holder

## Definition
```java
// Signature
public class WorldPathData implements Resource<EntityStore> {
```

## Architecture & Concepts
WorldPathData is a server-side data component that acts as a container for all AI navigation path information within a specific world context. It implements the Resource interface, indicating that its lifecycle is not managed directly but is instead attached to and owned by a parent `EntityStore`.

This class does not represent a global pathing system. Instead, each instance holds path data for a single `EntityStore`, effectively scoping it to a specific world or dimension. Its primary architectural function is to organize and provide fast lookups for `IPrefabPath` objects, which define the traversable routes for NPCs and other entities.

Paths are further partitioned internally by a `worldgenId`. This allows the system to manage distinct sets of paths that may correspond to different world generation layers, dimensions, or zones within the same `EntityStore`. All path queries and modifications are routed through this class, making it the authoritative source for prefab-based pathing data on the server.

### Lifecycle & Ownership
- **Creation:** An instance of WorldPathData is never created directly via its constructor. It is lazily instantiated by the engine's resource management system when path data is first requested for a given `EntityStore`. The static `getResourceType` method provides the key for this lookup and creation mechanism.

- **Scope:** The lifecycle of a WorldPathData instance is strictly bound to its parent `EntityStore`. It persists in memory as long as the corresponding world or dimension is loaded and active on the server.

- **Destruction:** The object is eligible for garbage collection when its parent `EntityStore` is unloaded from memory. There is no explicit destruction or cleanup method; its fate is tied entirely to its owner.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its core state is the `prefabPaths` map, which is a live cache of all known navigation paths. This map is continuously modified during gameplay as new paths are discovered, waypoints are loaded or unloaded, and old paths are removed.

- **Thread Safety:** **CRITICAL WARNING:** This class is **not thread-safe**. The internal data structure, `Int2ObjectOpenHashMap`, is not designed for concurrent access. All method calls that read from or write to this object **must** be externally synchronized. In the Hytale server architecture, this synchronization is expected to be handled by the main server thread that owns the parent `EntityStore`. Accessing an instance from asynchronous jobs or worker threads without explicit locking will result in `ConcurrentModificationException` or silent data corruption.

## API Surface
The public API provides methods for querying, creating, and removing pathing information.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNearestPrefabPath(...) | IPrefabPath | O(N) | Finds the closest path to a given position. Complexity is linear to the number of paths in the specified worldgen collection. |
| getOrConstructPrefabPath(...) | IPrefabPath | O(1) | Retrieves a path by its UUID or creates it using a provided generator function if it does not exist. Average case complexity is constant time. |
| removePrefabPath(worldgenId, id) | void | O(1) | Deletes a path and its associated collection if the collection becomes empty. |
| unloadPrefabPathWaypoint(...) | void | O(1) | Marks a specific waypoint as unloaded, potentially affecting the path's load state. |
| getPrefabPath(worldgenId, id, ignoreLoadState) | IPrefabPath | O(1) | Retrieves a path by its UUID. Can optionally return paths that are not fully loaded. |
| getAllPrefabPaths() | List<IPrefabPath> | O(N*M) | **Expensive Operation.** Creates and returns a new unmodifiable list of every path across all worldgen collections. Avoid calling this on a frequent basis. |

## Integration Patterns

### Standard Usage
WorldPathData should always be retrieved from a `ComponentAccessor` that has access to an `EntityStore`. This ensures you are operating on the correct data container for the given world context.

```java
// Assuming 'accessor' is a ComponentAccessor<EntityStore> for the target world
WorldPathData pathData = accessor.getOrCompute(WorldPathData.getResourceType());

// Find the nearest path for an entity at a specific position
IPrefabPath nearest = pathData.getNearestPrefabPath(worldgenId, entityPosition, disallowedPaths, accessor);

if (nearest != null) {
    // Begin pathfinding logic...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldPathData()`. This creates an orphan object that is not managed by the engine's resource system, leading to memory leaks and a disconnected state.
- **Cross-thread Access:** Do not access a WorldPathData instance from a separate worker thread without acquiring a lock on the parent `EntityStore` or its managing thread. This is the most common cause of server instability related to this system.
- **State Caching:** Do not call `getAllPrefabPaths()` and cache the result for an extended period. The internal state of WorldPathData is volatile, and the cached list will quickly become stale.

## Data Pipeline
WorldPathData primarily serves as a stateful repository rather than a processing stage in a pipeline. Data flows into it upon creation and is queried from it by various game systems.

> **Path Creation Flow:**
> World Generation Event -> Prefab Placement -> `PathGenerator` -> **WorldPathData.getOrConstructPrefabPath** -> Stored in `prefabPaths` map

> **Path Query Flow:**
> AI Behavior System -> **WorldPathData.getNearestPrefabPath** -> Pathfinding Algorithm -> Entity Movement

