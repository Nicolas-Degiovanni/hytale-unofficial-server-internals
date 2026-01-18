---
description: Architectural reference for PrefabPathCollection
---

# PrefabPathCollection

**Package:** com.hypixel.hytale.builtin.path
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPathCollection {
```

## Architecture & Concepts
The PrefabPathCollection is a stateful container responsible for managing the lifecycle and facilitating the discovery of all IPrefabPath instances within a specific world generation context. It functions as a scoped, in-memory repository on the server, indexed by a unique `worldgenId`.

This class is not a global system but rather a component of a larger world or zone entity. Its primary architectural purpose is to provide an optimized query interface for systems, particularly AI, that need to perform spatial searches for navigation paths.

To achieve efficient lookups, it maintains two internal data structures:
1.  A primary `Map<UUID, IPrefabPath>` for direct, constant-time retrieval of a path by its unique identifier.
2.  A secondary `Int2ObjectMap<PathSet>` which groups paths by a "friendly name". This name is converted to an integer index via the global AssetRegistry, a common engine pattern for string interning and fast lookups. This secondary index is critical for querying paths of a specific type (e.g., "patrol_route") without iterating through the entire collection.

This dual-indexing strategy allows the system to support both direct manipulation of specific path instances and broader, type-based spatial queries.

## Lifecycle & Ownership
-   **Creation:** A PrefabPathCollection is instantiated by a higher-level world management system, such as a `World` or `Zone` object, when that region is loaded or generated. The `worldgenId` provided during construction permanently associates the collection with its parent world context.
-   **Scope:** The object's lifetime is strictly bound to its parent world container. It persists in memory for as long as the corresponding world zone is active on the server.
-   **Destruction:** The class has no explicit `destroy` or `close` method. It is marked for garbage collection when the parent `World` or `Zone` object is unloaded and all references to the collection are released. Failure to properly manage the lifecycle of the parent container will result in this collection leaking memory.

## Internal State & Concurrency
-   **State:** This class is highly mutable. Its internal maps are continuously modified as paths and waypoints are added, removed, or unloaded during gameplay and world-generation events. It effectively serves as a live cache of pathing data for a server-side world region.
-   **Thread Safety:** **This class is not thread-safe.** The internal collections are non-concurrent `fastutil` maps, chosen for performance. All read and write operations must be synchronized with the main server thread (the game tick). Accessing this object from asynchronous tasks or other threads without external locking will lead to `ConcurrentModificationException`, data corruption, and unpredictable server behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNearestPrefabPath(nameIndex, position, ...) | IPrefabPath | O(N * W) | Finds the closest path of a specific type. N is the number of paths with the given name; W is the average waypoint count. |
| getNearestPrefabPath(position, ...) | IPrefabPath | O(P * W) | Finds the closest path of any type by performing a full scan. P is the total number of paths in the collection. |
| getOrConstructPath(id, name, pathGenerator) | IPrefabPath | O(1) avg | Atomically retrieves a path by UUID or creates it using the provided generator if it does not exist. This is the primary method for populating the collection. |
| removePathWaypoint(id, index) | void | O(1) avg | Removes a waypoint from a path. If the path becomes empty, the entire path is automatically removed from the collection. |
| unloadPathWaypoint(id, index) | void | O(1) avg | Marks a waypoint as unloaded. If a path has no loaded waypoints left, it is removed from the collection. |
| removePath(id) | void | O(1) avg | Explicitly removes an entire path and all its waypoints from the collection. |

## Integration Patterns

### Standard Usage
A server-side AI or quest system retrieves the collection from its world context and uses it to find a relevant path for an entity.

```java
// Assume 'world' is the current server world object
// Assume 'aiPosition' is the entity's current location

// Get the path collection for the current world context
PrefabPathCollection pathCollection = world.getPathCollection();

// Find the nearest path named "guard_patrol"
int patrolNameIndex = AssetRegistry.getTagIndex("guard_patrol");
IPrefabPath patrolPath = pathCollection.getNearestPrefabPath(patrolNameIndex, aiPosition, null, world.getComponentAccessor());

if (patrolPath != null) {
    // AI can now use the patrolPath for navigation
    aiController.followPath(patrolPath);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new PrefabPathCollection()`. The collection is managed by the world system and must be retrieved from the active world context. Direct instantiation creates an orphan object that is not integrated with the game state.
-   **State Caching:** Do not cache the result of `getNearestPrefabPath` over long periods. The collection is dynamic; paths can be removed or altered at any time. Re-query the collection when a new path-finding decision is required.
-   **Cross-Thread Modification:** Never call methods like `removePath` or `getOrConstructPath` from a worker thread. All modifications must be scheduled to run on the main server game thread.

## Data Pipeline
PrefabPathCollection acts as a repository, not a pipeline stage. Data flows into it during world generation or dynamic events, and it is queried on-demand by other systems.

> **Ingress Flow (Path Creation):**
> World Generator -> `getOrConstructPath` -> **PrefabPathCollection** -> (Stores new IPrefabPath)

> **Query Flow (Path Usage):**
> AI Behavior System -> `getNearestPrefabPath` -> **PrefabPathCollection** (Performs spatial search) -> Returns IPrefabPath -> AI Behavior System

