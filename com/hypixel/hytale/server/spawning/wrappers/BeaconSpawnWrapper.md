---
description: Architectural reference for BeaconSpawnWrapper
---

# BeaconSpawnWrapper

**Package:** com.hypixel.hytale.server.spawning.wrappers
**Type:** Transient Data Wrapper

## Definition
```java
// Signature
public class BeaconSpawnWrapper extends SpawnWrapper<BeaconNPCSpawn> {
```

## Architecture & Concepts
The BeaconSpawnWrapper is a runtime adapter that translates a static data asset, BeaconNPCSpawn, into an optimized, queryable object for the server's NPC spawning system. Its primary purpose is to encapsulate the complex rules and probabilities of a single beacon-based spawning configuration and present them through a high-performance interface.

This class is not merely a data container. It performs critical pre-processing during its construction:
1.  **Weighted Role Selection:** It consumes a list of potential NPC roles and their spawn weights from the asset and builds an IWeightedMap. This internal data structure allows for efficient, O(log N) probabilistic selection of which NPC to spawn.
2.  **Performance Optimization:** It pre-calculates the squared values of player distance thresholds (min and target). This is a standard optimization that allows the spawning system to perform distance checks using squared-euclidean distance, avoiding expensive square root calculations in the game loop.

In essence, this wrapper acts as a performance-oriented bridge between the static asset configuration and the dynamic, high-frequency logic of the server's spawning engine.

## Lifecycle & Ownership
-   **Creation:** A BeaconSpawnWrapper is instantiated by a higher-level spawning manager or factory. This typically occurs when a BeaconNPCSpawn asset is loaded from disk and needs to be made active for a specific region or game event. The constructor is the sole entry point.
-   **Scope:** The object's lifetime is tied to the system that requires its specific spawning rules. It is not a global singleton. It is expected to be cached by a zone or region manager for as long as the associated beacon type is relevant, then discarded.
-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and has no explicit `destroy` or `close` method. It becomes eligible for collection once all references from the spawning system are released.

## Internal State & Concurrency
-   **State:** The BeaconSpawnWrapper is effectively **immutable**. All internal fields, including the weightedRoles map and the pre-calculated squared distances, are marked as final and are exclusively populated within the constructor. The object's state is a direct, cached transformation of the source BeaconNPCSpawn asset and does not change during its lifetime.

-   **Thread Safety:** This class is **conditionally thread-safe**.
    -   **Read Operations:** All getter methods and internal state are safe to be read from multiple threads concurrently without external synchronization.
    -   **`pickRole` Method:** The safety of this method depends entirely on the provided Random instance. If multiple threads call `pickRole` using a shared, non-thread-safe Random object (like java.util.Random), external synchronization is **required** to prevent race conditions within the random number generator. The BeaconSpawnWrapper itself performs no internal locking.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| pickRole(Random chanceProvider) | RoleSpawnParameters | O(log N) | Probabilistically selects an NPC role based on configured weights. Returns null if no valid roles are available. |
| getMinDistanceFromPlayerSquared() | double | O(1) | Returns the pre-calculated squared minimum distance a player must be from the spawn location. |
| getTargetDistanceFromPlayerSquared() | double | O(1) | Returns the pre-calculated squared ideal distance a player should be from the spawn location. |
| getBeaconRadius() | double | O(1) | Returns the radius of the beacon's influence. |
| getSpawnRadius() | double | O(1) | Returns the radius within which NPCs can be spawned around the beacon. |

## Integration Patterns

### Standard Usage
The intended use is for a spawner service to create or retrieve a cached instance of this wrapper corresponding to a specific beacon asset. During a spawn tick, the service calls `pickRole` to determine which NPC to potentially spawn.

```java
// Spawning system retrieves a wrapper for a given beacon type
BeaconSpawnWrapper beaconRules = SpawnerRegistry.getWrapperFor("undead_crypt_beacon");

// A thread-local or per-tick random instance is recommended
Random random = new Random(); 

// Decide which NPC to spawn based on weighted chance
RoleSpawnParameters chosenRole = beaconRules.pickRole(random);

if (chosenRole != null) {
    // Proceed with spawning logic for the chosen role...
}
```

### Anti-Patterns (Do NOT do this)
-   **Per-Tick Instantiation:** Do not create a new BeaconSpawnWrapper on every spawn check. The constructor is a relatively expensive operation that builds the weighted map. These objects are designed to be created once and reused.
    ```java
    // ANTI-PATTERN: Inefficiently creating a new wrapper in a loop
    for (Beacon b : activeBeacons) {
        BeaconSpawnWrapper wrapper = new BeaconSpawnWrapper(b.getSpawnAsset()); // BAD
        wrapper.pickRole(random);
    }
    ```
-   **Shared Non-Thread-Safe Random:** Do not pass a single, shared java.util.Random instance to `pickRole` from multiple concurrent spawning threads without external locking. This will lead to thread contention and unpredictable results. Use a ThreadLocalRandom or a properly synchronized generator instead.

## Data Pipeline
The BeaconSpawnWrapper is a key transformation step in the server spawning data flow. It converts static configuration data into a usable, high-performance runtime object.

> Flow:
> BeaconNPCSpawn Asset (JSON) -> Asset Manager -> **BeaconSpawnWrapper** (Constructor builds weighted map) -> Spawning System (calls `pickRole`) -> RoleSpawnParameters -> NPC Entity Factory

