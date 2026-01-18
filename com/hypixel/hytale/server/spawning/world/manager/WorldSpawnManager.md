---
description: Architectural reference for WorldSpawnManager
---

# WorldSpawnManager

**Package:** com.hypixel.hytale.server.spawning.world.manager
**Type:** Singleton

## Definition
```java
// Signature
public class WorldSpawnManager extends SpawnManager<WorldSpawnWrapper, WorldNPCSpawn> {
```

## Architecture & Concepts

The WorldSpawnManager is the central authority for NPC spawning rules within the server. It acts as a registry and rule-engine that maps NPC types, environmental conditions, and specific spawn configurations. Its primary role is to process and validate `WorldNPCSpawn` asset data, making it available to the per-world spawning systems.

This manager is not responsible for the act of spawning itself (which is handled by systems processing `SpawnJobData` components). Instead, it maintains the high-level strategic state: what *can* spawn, *where*, and at what *density*. It directly manipulates the `WorldSpawnData` resource within each world's Entity Component System (ECS) store, ensuring that runtime spawn counters and configurations are synchronized with the loaded asset data.

It operates at a global scope, bridging the gap between the asset pipeline and the live state of every running `World` instance in the `Universe`. Changes made to this manager, such as rebuilding configurations, have immediate and server-wide effects.

## Lifecycle & Ownership

-   **Creation:** A single instance of WorldSpawnManager is created and owned by the `SpawningPlugin` during server initialization. It is not instantiated on a per-world basis.
-   **Scope:** The instance is a session-scoped singleton. It persists for the entire lifecycle of the server process.
-   **Destruction:** The manager and its cached data are destroyed when the `SpawningPlugin` is unloaded, typically during a server shutdown sequence.

## Internal State & Concurrency

-   **State:** The WorldSpawnManager is highly stateful and mutable. It maintains several internal maps that cache processed spawn rules derived from game assets.
    -   `environmentSpawnParametersMap`: Caches spawn density and associated configurations for each environment ID.
    -   `npcEnvCombinations`: A critical lookup table to enforce the rule that a specific NPC and environment combination can only be defined by one spawn configuration. This prevents ambiguity and data corruption.
    -   `npcTypesPerEnvironment`: A quick-access map to retrieve all valid NPC roles for a given environment.

-   **Thread Safety:** This class is **not** thread-safe.
    -   While it uses an `Int2ObjectConcurrentHashMap` for one of its primary caches, the broader logic involving multiple collections is not synchronized.
    -   Modification methods like `addSpawnWrapper`, `removeSpawnWrapper`, and `rebuildConfigurations` perform multi-step operations that are not atomic and must be called from a single, controlled thread (e.g., the main server thread).
    -   Operations that modify world state are safely dispatched to the respective world's execution thread via the `world.execute()` pattern. However, this does not protect the manager's own internal state from concurrent modification.

    **WARNING:** Do not invoke state-modifying methods on this manager from multiple threads without external, system-level locking. Doing so will lead to a corrupted spawning state, race conditions, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addSpawnWrapper(WorldSpawnWrapper) | boolean | High | Registers a new spawn configuration. Validates against existing rules and updates all running worlds. |
| removeSpawnWrapper(int) | WorldSpawnWrapper | High | Deregisters a spawn configuration by its index, cleans up its rules, and updates all running worlds. |
| rebuildConfigurations(IntSet) | void | Very High | Atomically removes and re-adds a set of spawn configurations. This is the primary entry point for hot-reloading asset changes. |
| getRolesForEnvironment(int) | IntSet | O(1) | Returns the set of NPC role indices that are configured to spawn in the given environment. |
| trackNPCs(IntSet) | static void | O(W*E) | Scans all entities (E) in all worlds (W) to update spawn tracking counts for the given configurations. Extremely expensive. |
| untrackNPCs(IntSet) | static void | O(W*E) | Scans all entities in all worlds to remove them from spawn tracking counts. Extremely expensive. |

## Integration Patterns

### Standard Usage

The WorldSpawnManager is typically controlled by higher-level systems, such as an asset hot-reloading service. Direct interaction should be limited to initialization and data update cycles.

```java
// Example: Responding to an asset update event
public void onSpawnAssetsChanged(IntSet changedConfigIndices) {
    SpawningPlugin plugin = SpawningPlugin.get();
    WorldSpawnManager manager = plugin.getWorldSpawnManager();

    // This single call handles all logic for removing old rules
    // and adding the new ones for the specified configurations.
    manager.rebuildConfigurations(changedConfigIndices);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new WorldSpawnManager()`. The singleton instance must be retrieved from the `SpawningPlugin`. An orphaned instance will not be integrated with the game's worlds or asset systems.
-   **Concurrent Modification:** Do not call `addSpawnWrapper` or `rebuildConfigurations` from worker threads without a robust locking mechanism. This will corrupt the manager's internal state.
-   **Frequent Rebuilds:** Avoid calling `rebuildConfigurations` on a frequent basis (e.g., per-frame or per-second). It is a heavy operation designed to handle asset updates, which are infrequent.
-   **Excessive Tracking Calls:** The static `trackNPCs` and `untrackNPCs` methods iterate over every entity on the server. They are designed for bulk processing during configuration reloads, not for tracking individual entity spawns or deaths.

## Data Pipeline

The primary data flow for this manager involves processing static asset data and propagating it into the live ECS state of each game world.

> Flow:
> Asset File (`.json`) -> Asset Loader -> `WorldNPCSpawn` Object -> **WorldSpawnManager**.`rebuildConfigurations()` -> Internal Rule Caches -> `World`.`execute()` -> `WorldSpawnData` Resource Update

