---
description: Architectural reference for BeaconSpawnManager
---

# BeaconSpawnManager

**Package:** com.hypixel.hytale.server.spawning.managers
**Type:** Singleton

## Definition
```java
// Signature
public class BeaconSpawnManager extends SpawnManager<BeaconSpawnWrapper, BeaconNPCSpawn> {
```

## Architecture & Concepts

The BeaconSpawnManager is a specialized, server-side system responsible for indexing and managing NPC spawn configurations that are tied to in-world "beacons". It extends the generic functionality of its parent, SpawnManager, by adding a critical layer of indexing based on environment IDs.

Its primary architectural role is to provide an optimized lookup mechanism for the core spawning system. When the server needs to determine which NPCs can spawn in a particular area (e.g., a specific biome, dungeon, or dimension), it queries this manager using an environment ID. The manager returns a pre-compiled list of all valid beacon-related spawns for that context, dramatically accelerating the spawn evaluation process.

This class effectively acts as a live, in-memory cache that maps an abstract environmental context to a concrete set of spawnable entities. It decouples the spawning logic from the need to iterate through all possible spawn configurations, providing a significant performance gain.

## Lifecycle & Ownership

-   **Creation:** Instantiated once per server or world instance during the server's main bootstrap sequence. It is created and managed by a higher-level service container or world context, not directly by game logic.
-   **Scope:** Session-scoped. The BeaconSpawnManager persists for the entire lifetime of the server session or world it belongs to. Its internal state is built up as spawn configurations are loaded and registered.
-   **Destruction:** The manager and its cached data are destroyed during the server shutdown or world unloading process. There is no public API for manual destruction; its lifecycle is entirely managed by the engine.

## Internal State & Concurrency

-   **State:** The BeaconSpawnManager is highly stateful and mutable. Its core state is maintained in the **wrappersByEnvironment** field, an `Int2ObjectConcurrentHashMap`. This map caches lists of spawn wrappers, keyed by their associated environment IDs. The state changes dynamically as beacon spawns are added or removed during gameplay.

-   **Thread Safety:** This component has nuanced thread-safety characteristics that require careful consideration.
    -   The top-level map, `Int2ObjectConcurrentHashMap`, is thread-safe for map-level operations like adding a new environment entry (`computeIfAbsent`) or retrieving a list for an existing environment (`get`).
    -   **WARNING:** The value stored in the map, a `List<BeaconSpawnWrapper>` (specifically an `ObjectArrayList`), is **not thread-safe**.
    -   The `addSpawnWrapper` and `removeSpawnWrapper` methods modify these internal lists. If multiple threads attempt to modify the list for the *same environment ID* concurrently, it will lead to race conditions, `ConcurrentModificationException`, or data corruption.
    -   All modifications to a specific environment's spawn list must be externally synchronized or guaranteed to execute on a single, dedicated thread (e.g., the main server tick thread).

## API Surface

The public API is designed for registering, unregistering, and querying spawn configurations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addSpawnWrapper(BeaconSpawnWrapper) | boolean | O(k) | Registers a new spawn wrapper. Indexes it against all *k* environment IDs it specifies. |
| removeSpawnWrapper(int) | BeaconSpawnWrapper | O(k) | Removes a spawn wrapper by its configuration index. De-indexes it from all *k* environments. |
| getBeaconSpawns(int) | List | O(1) | Retrieves the list of all registered beacon spawns for a given environment ID. |

## Integration Patterns

### Standard Usage

The manager should always be retrieved from the central server context. The primary interaction pattern is to query for spawns based on the current player or world environment.

```java
// Correctly retrieve the manager from the server context
BeaconSpawnManager spawnManager = serverContext.get(BeaconSpawnManager.class);

// Get the current environment ID from the world or player state
int currentEnvironmentId = world.getEnvironmentAt(playerPosition);

// Query for all valid spawns in that environment
List<BeaconSpawnWrapper> validSpawns = spawnManager.getBeaconSpawns(currentEnvironmentId);

// The spawning system can now process this filtered list
if (validSpawns != null) {
    for (BeaconSpawnWrapper spawn : validSpawns) {
        // Evaluate spawn conditions...
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new BeaconSpawnManager()`. This will result in a disconnected, unmanaged object that is not integrated with the server's lifecycle or data sources.
-   **Modifying Returned Lists:** The list returned by `getBeaconSpawns` is a direct reference to the internal cache, not a copy. Modifying this list will corrupt the manager's state and cause unpredictable behavior in the spawning system. Treat the returned list as read-only.
-   **Concurrent Modification:** Do not call `addSpawnWrapper` or `removeSpawnWrapper` for the same environment from multiple threads without implementing external locking. This will break the internal list's integrity.

## Data Pipeline

The BeaconSpawnManager serves as a crucial indexing step in the server's spawn configuration data pipeline. It transforms a flat stream of spawn data into a structured, queryable cache.

> Flow:
> Game Asset (BeaconNPCSpawn.json) -> Asset Deserializer -> **BeaconSpawnManager.addSpawnWrapper** -> Internal Environment-Keyed Map -> Spawning System Query (getBeaconSpawns) -> Spawn Evaluation Logic

