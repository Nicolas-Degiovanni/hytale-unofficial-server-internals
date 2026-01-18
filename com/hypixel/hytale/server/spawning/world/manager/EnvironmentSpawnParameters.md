---
description: Architectural reference for EnvironmentSpawnParameters
---

# EnvironmentSpawnParameters

**Package:** com.hypixel.hytale.server.spawning.world.manager
**Type:** Transient

## Definition
```java
// Signature
public class EnvironmentSpawnParameters {
```

## Architecture & Concepts
The EnvironmentSpawnParameters class is a stateful data container that encapsulates the rules for entity spawning within a specific server-side environment, such as a biome or a defined world zone. It does not contain any logic itself; instead, it serves as a parameter object passed to the core spawning managers which interpret its data to execute spawn ticks.

This class holds two fundamental pieces of information:
1.  **Spawn Density:** A quantitative measure controlling the overall population pressure or frequency of spawn attempts in an area.
2.  **Spawn Wrappers:** A qualitative collection defining the specific types of entities eligible to spawn in the environment.

It is designed to be a mutable configuration object, allowing for dynamic changes to spawning behavior at runtime, for example, in response to in-game events, time of day, or player actions.

## Lifecycle & Ownership
-   **Creation:** Instances are typically created by higher-level world configuration systems when a game zone or biome is loaded and its properties are parsed. It is not a globally managed service but rather a component of a larger configuration structure.
-   **Scope:** The lifetime of an EnvironmentSpawnParameters object is tied directly to the lifetime of its owning context. For example, an instance representing a specific biome's rules will persist as long as that biome definition is loaded in server memory.
-   **Destruction:** The object is eligible for garbage collection when its parent configuration context is unloaded or discarded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This is a **highly mutable** object. Both the spawn density and the set of spawn wrappers can be modified after instantiation.

-   **Thread Safety:** The class exhibits mixed thread-safety, which is a critical design detail requiring careful handling.
    -   The **density** field is **not thread-safe**. Access and modification via getSpawnDensity and setDensity are unsynchronized. Concurrent writes from different threads will result in a race condition, and visibility of updates is not guaranteed without external synchronization.
    -   The **spawnWrappers** set **is thread-safe**. It is backed by a ConcurrentHashMap.newKeySet(), which permits concurrent additions, removals, and traversals without external locking. This is designed to allow systems to dynamically alter the list of potential spawns (e.g., for a special event) without disrupting read operations from the main spawner loop.

    **WARNING:** Due to the unsynchronized nature of the density field, any system that modifies an instance of this class must ensure it is not being concurrently read or written by another thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnWrappers() | Set<WorldSpawnWrapper> | O(1) | Returns a thread-safe view of the set of spawnable entities. |
| getSpawnDensity() | double | O(1) | Retrieves the current spawn density. Not synchronized. |
| setDensity(density) | void | O(1) | Updates the spawn density. Not synchronized. |

## Integration Patterns

### Standard Usage
An instance is typically configured by a zone loader and then held by a manager responsible for a specific world area. The core spawning system reads from this object on each spawn tick.

```java
// A world manager configures spawn parameters for a new forest biome
EnvironmentSpawnParameters forestParams = new EnvironmentSpawnParameters(0.75);
forestParams.getSpawnWrappers().add(new WorldSpawnWrapper(/* Deer */));
forestParams.getSpawnWrappers().add(new WorldSpawnWrapper(/* Wolf */));

// The biome manager now holds this configuration
loadedBiome.setSpawnParameters(forestParams);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Density Modification:** Do not allow multiple threads to call setDensity on a shared instance without external locking. This will lead to unpredictable density values.
-   **Misinterpreting Thread Safety:** Do not assume the entire object is thread-safe just because the wrapper set is. The density field must be treated as thread-unsafe.
-   **Stale Reads:** Do not read the density, perform a long-running operation, and then use that value. The density may have been changed by another system in the interim. Re-fetch the value if it must be current.

## Data Pipeline
This class acts as a source of configuration data for the entity spawning pipeline. It does not process data itself.

> Flow:
> World Configuration Files -> Zone/Biome Manager -> **EnvironmentSpawnParameters** -> WorldSpawnManager -> Entity Spawning Logic

