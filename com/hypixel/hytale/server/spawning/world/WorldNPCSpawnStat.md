---
description: Architectural reference for WorldNPCSpawnStat
---

# WorldNPCSpawnStat

**Package:** com.hypixel.hytale.server.spawning.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldNPCSpawnStat {
```

## Architecture & Concepts

The WorldNPCSpawnStat class is a stateful data object that acts as a runtime "scorecard" for a single NPC role within a specific World's spawning system. It is not a service or a manager; rather, it is a granular tracking mechanism used by higher-level spawning controllers to make informed decisions.

For each unique NPC role that can be spawned in a world, a corresponding WorldNPCSpawnStat instance is created. This object's primary responsibility is to maintain the critical delta between the *desired* population of an NPC role (the expected count) and its *current* population (the actual count).

The server's main spawning loop iterates over a collection of these objects. By querying methods like getMissingCount and getWeight, the system determines which NPCs to prioritize for spawning. This class also serves as a vital diagnostics tool, aggregating detailed metrics about spawn attempts, successes, failures, and the specific reasons for rejection.

A key architectural feature is its use of a WeakReference to the NPC's BuilderInfo. This prevents the stat object from holding a strong reference to potentially large asset-related data, allowing the garbage collector to reclaim memory for NPC roles that are no longer in use or have been unloaded. The class is designed to gracefully handle the absence of this data by re-querying the central NPCPlugin when necessary.

The specialized inner class, CountOnly, implements a variation of the Null Object pattern. It is used in scenarios where the system only needs to track the *actual* count of an NPC role without considering any spawning logic, effectively disabling spawning calculations for that instance.

## Lifecycle & Ownership

-   **Creation:** Instances are created and owned by a world-specific spawning manager, such as a WorldSpawnManager. One instance is typically instantiated for each configured RoleSpawnParameters associated with the world.
-   **Scope:** The object's lifetime is tightly coupled to the spawning context of its parent World. It persists as long as the managing spawner is active and the associated NPC role is considered a valid spawn candidate. The internal data is mutable and continuously updated during the server's spawn ticks.
-   **Destruction:** The object is eligible for garbage collection when the managing spawner is destroyed (e.g., during a world unload) or when it rebuilds its list of spawnable roles and the role tracked by this instance is no longer valid.

## Internal State & Concurrency

-   **State:** This object is highly mutable. Its core purpose is to have its internal counters and state flags (actual, expected, spansTried, unspawnable) modified by the single, authoritative spawning system that owns it. It holds direct references to its world and spawn configuration, and a weak reference to asset-related builder information.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is designed with the strict assumption that it will be created, manipulated, and read by a single thread—the server's main world-processing thread.

    **WARNING:** Any attempt to access or modify a WorldNPCSpawnStat instance from a concurrent thread without external, robust locking mechanisms will result in race conditions, data corruption, and unpredictable spawning behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMissingCount(accessor) | double | O(1) Amortized | Calculates the number of NPCs needed to reach the expected count. Returns 0.0 if unspawnable or if conditions are not met. May trigger an expensive lookup if the internal weak reference has been cleared. |
| updateSpawnStats(...) | void | O(1) | The primary mutation method. Updates all internal counters after a spawn job has been attempted. This is the main entry point for feeding diagnostic data into the object. |
| getWeight(moonPhase) | double | O(1) | Returns the spawn "pressure" or priority for this role, adjusted for the current moon phase. Used by the spawner to weigh spawn candidates. |
| adjustActual(count) | void | O(1) | Directly modifies the internal count of actual NPCs in the world. Called when NPCs of this role are spawned or despawned. |
| setExpected(expected) | void | O(1) | Sets the target population for this NPC role. This is typically calculated by the spawning system based on player density, location, and other world state. |
| isUnspawnable() | boolean | O(1) | A flag indicating if this role has been marked as impossible to spawn, short-circuiting further spawn attempts. |
| CountOnly(roleIndex) | constructor | O(1) | Creates a lightweight instance that only tracks the actual count and disables all spawning logic (getMissingCount and getWeight always return 0). |

## Integration Patterns

### Standard Usage

The WorldNPCSpawnStat object is intended to be managed by a higher-level controller. The controller is responsible for the "macro" logic of calculating expected counts and the "micro" logic of updating stats after each spawn attempt.

```java
// In a hypothetical WorldSpawnManager during its update tick:
// 1. Create and maintain a map of roleIndex -> WorldNPCSpawnStat
Map<Integer, WorldNPCSpawnStat> stats = this.getStatsForWorld(world);

// 2. The manager calculates the desired population for a role
double desiredPopulation = calculateExpectedNpcsForRole(role, world);
WorldNPCSpawnStat stat = stats.get(role.getIndex());
stat.setExpected(desiredPopulation);

// 3. The manager queries the stat object to decide if a spawn is needed
double missingCount = stat.getMissingCount(worldEntityAccessor);
if (missingCount > 0) {
    // 4. Attempt to spawn the NPC...
    SpawnResult result = attemptToSpawn(role, world);

    // 5. Feed the results back into the stat object for diagnostics
    stat.updateSpawnStats(
        result.getSpansTried(),
        result.getSpansSuccess(),
        result.getBudgetUsed(),
        result.getRejections(),
        result.wasSuccessful()
    );
}
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Do not read or write to a WorldNPCSpawnStat instance from multiple threads. All interactions must be synchronized externally or, preferably, confined to the single world thread.
-   **State Desynchronization:** Do not hold onto instances across major world state changes (e.g., asset reloads) without re-validating their state. The internal weak reference to BuilderInfo is designed to handle this, but other references like WorldSpawnWrapper may become stale.
-   **Misusing CountOnly:** Do not use the CountOnly subclass for roles that are expected to be actively spawned. Its purpose is purely for tracking existing populations where spawning logic is irrelevant or explicitly disabled.

## Data Pipeline

The WorldNPCSpawnStat object acts as a stateful node within the server's spawning data pipeline. It does not process data itself, but rather has data written to it and read from it at different stages of the spawn tick.

> Flow:
> Spawning System Tick -> Calculate Desired Population -> **WorldNPCSpawnStat.setExpected()** -> Spawner Weighs Candidates -> **WorldNPCSpawnStat.getWeight()** -> Spawner Selects Candidate -> **WorldNPCSpawnStat.getMissingCount()** -> Spawn Job Execution -> Spawn Result -> **WorldNPCSpawnStat.updateSpawnStats()** -> Server Diagnostics & Logging

