---
description: Architectural reference for SpawnManager
---

# SpawnManager

**Package:** com.hypixel.hytale.server.spawning.managers
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class SpawnManager<T extends SpawnWrapper<U>, U extends NPCSpawn> {
```

## Architecture & Concepts

The SpawnManager is an abstract base class that provides the core, thread-safe caching and management logic for all server-side NPC spawn configurations. It is not used directly but is extended by concrete implementations to manage specific categories of spawns, such as those for world generation or points of interest.

Architecturally, this class serves as a high-performance, in-memory data store that decouples the game's spawning systems from the raw asset configuration files. Its primary responsibility is to maintain a consistent and rapidly accessible cache of processed SpawnWrapper objects, which contain runtime-relevant data derived from NPCSpawn assets.

Key design principles include:

*   **Concurrency:** The class is built for a highly concurrent server environment. It employs a StampedLock to ensure that multiple game threads can read spawn data simultaneously without blocking, while write operations (like adding or removing a spawn) maintain data integrity.
*   **Reactivity:** SpawnManager is designed to react to live changes in the server's asset system. Methods like onNPCLoaded and onNPCSpawnRemoved act as event handlers, allowing the manager to dynamically update its cache when NPC or spawn assets are loaded, reloaded, or removed. This is critical for supporting server hot-reloading and dynamic content updates.
*   **Performance:** The use of fastutil collections (Int2ObjectOpenHashMap, Object2IntOpenHashMap) and dual-mapping (index-to-wrapper and name-to-index) ensures that lookups are, on average, O(1) operations.

## Lifecycle & Ownership

-   **Creation:** Concrete subclasses of SpawnManager are instantiated by the SpawningPlugin during server initialization. They are not intended for direct instantiation by game logic code.
-   **Scope:** A SpawnManager instance persists for the entire lifetime of the server session. Its internal cache is populated as assets are loaded and is cleared upon server shutdown.
-   **Destruction:** The manager and its cached data are garbage collected during the server shutdown sequence when the SpawningPlugin is disabled.

## Internal State & Concurrency

-   **State:** The SpawnManager is a highly mutable, stateful component. Its core state consists of the spawnWrapperCache and wrapperNameMap, which are continuously modified at runtime as spawn configurations are loaded and unloaded by the asset system.
-   **Thread Safety:** This class is thread-safe. All read and write operations on the internal caches are strictly controlled by a StampedLock. This sophisticated locking mechanism allows for high-throughput, non-blocking reads from multiple threads while ensuring that write operations are atomic and exclusive.

**WARNING:** While the manager's methods are thread-safe, the SpawnWrapper objects it returns are not guaranteed to be. Modifications to a retrieved wrapper must be handled with care by the calling system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnWrapper(int spawnConfigIndex) | T | O(1) | Retrieves a cached SpawnWrapper by its unique configuration index. Returns null if not found. |
| addSpawnWrapper(T spawnWrapper) | boolean | O(1) | Adds a new SpawnWrapper to the cache. This is a write-locked operation. |
| removeSpawnWrapper(int spawnConfigIndex) | T | O(1) | Removes a SpawnWrapper from the cache by its index. Returns the removed wrapper or null. |
| onNPCLoaded(String name, IntSet changeSet) | void | O(N) | Scans all cached spawns to find those dependent on a newly loaded NPC asset. Populates the changeSet with indices of affected spawns. |
| onNPCSpawnRemoved(String key) | void | O(1) | Reacts to the removal of a spawn asset by its string key, triggering its removal from the cache. |

## Integration Patterns

### Standard Usage

The SpawnManager is not used directly. Other server systems interact with a concrete implementation, typically retrieved from the central SpawningPlugin. The primary use case is to query for spawn data needed by a spawner entity or system.

```java
// Example: A system reacting to spawn configuration changes
// Note: This is a hypothetical concrete implementation
WorldSpawnManager spawnManager = SpawningPlugin.get().getWorldSpawnManager();
IntSet changedSpawns = new IntOpenHashSet();

// Asset system notifies the manager of a change
spawnManager.onNPCLoaded("hytale:goblin_warrior", changedSpawns);

// The calling system now processes the invalidated spawns
for (int spawnIndex : changedSpawns) {
    // Logic to update or respawn entities associated with this index
}
```

### Anti-Patterns (Do NOT do this)

-   **External Iteration:** Do not attempt to iterate over the manager's internal collections from outside the class. This would bypass the locking mechanism and lead to severe concurrency issues, such as ConcurrentModificationException.
-   **Stale References:** Do not retrieve a SpawnWrapper and hold a reference to it for an extended period. Asset hot-reloading can cause a wrapper to be removed and replaced, leading to the use of stale or invalid data. Always re-fetch the wrapper from the manager when needed.
-   **Bypassing the Manager:** Do not read spawn configuration files directly. The SpawnManager and its wrappers contain processed, runtime-ready data that is not present in the raw assets. All spawn data must be accessed through the manager.

## Data Pipeline

The SpawnManager acts as a central hub in the data flow for spawn configurations, particularly during asset loading and hot-reloading events.

> Flow: Asset System Event (NPC asset loaded)
> 1.  The server's AssetManager finishes loading or reloading an NPC asset file (e.g., `goblin.json`).
> 2.  An event is dispatched to the SpawningPlugin.
> 3.  The SpawningPlugin invokes `onNPCLoaded("hytale:goblin", changeSet)` on the appropriate **SpawnManager**.
> 4.  The **SpawnManager** acquires a write lock and iterates its internal cache, identifying all SpawnWrapper instances that depend on the `hytale:goblin` NPC.
> 5.  The integer indices of these affected SpawnWrapper objects are added to the `changeSet`.
> 6.  The calling system receives the `changeSet` and can now take action, such as notifying relevant spawner entities to refresh their data.

