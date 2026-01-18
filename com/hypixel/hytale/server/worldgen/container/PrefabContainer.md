---
description: Architectural reference for PrefabContainer
---

# PrefabContainer

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Data Structure

## Definition
```java
// Signature
public class PrefabContainer {
```

## Architecture & Concepts

The PrefabContainer is a foundational, immutable data structure within the server-side world generation pipeline. It functions as a manifest or catalog that aggregates prefab placement rules for various environmental contexts.

Its primary architectural role is to decouple world generation logic from specific prefab data. Instead of hardcoding which structures can appear in a biome, the generator consults a PrefabContainer. This container holds a collection of **PrefabContainerEntry** objects, where each entry represents a distinct rule set. An entry links a weighted list of prefabs (e.g., small house, large tower) and a placement algorithm (a PrefabPatternGenerator) to a specific environment type, identified by an **environmentId**.

This design allows world generation to be highly data-driven. World designers can define complex prefab placement behavior in external configuration files, which are then loaded into PrefabContainer instances at runtime. The world generator remains generic, simply executing the rules defined within the container it is given for a particular region.

## Lifecycle & Ownership

-   **Creation:** PrefabContainer instances are not created dynamically during gameplay. They are instantiated by higher-level asset loaders or world initializers during the server's bootstrap phase. The constructor is typically fed data parsed from world generation configuration files (e.g., JSON assets).
-   **Scope:** An instance of PrefabContainer is long-lived. It persists for the entire server session, acting as a read-only configuration source for the world generation system.
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down or if the entire world generation configuration is reloaded, at which point the old instance would be dereferenced.

## Internal State & Concurrency

-   **State:** The class is designed to be effectively immutable. The root `entries` array and the pre-calculated `maxSize` are final. The nested PrefabContainerEntry objects also hold final references to their core data.
    -   **Lazy Initialization:** The `extend` field within PrefabContainerEntry is an exception. It is a lazily-initialized cache for the maximum size of the prefabs in that entry. This calculation is deferred until the first time `getExtents` is called.

-   **Thread Safety:** This class is conditionally thread-safe and intended for concurrent reads.
    -   The top-level PrefabContainer is thread-safe due to its immutability.
    -   The `getExtents` method in the nested PrefabContainerEntry class contains a benign race condition. The non-atomic check-and-set pattern (`if (extend == -1) { ... }`) means that if multiple world generation threads access it for the first time simultaneously, the extent calculation may be performed more than once. However, as the calculation is idempotent, the final state will be correct. This does not pose a data corruption risk but is a potential performance consideration under heavy contention.

    **WARNING:** The `getEntries` method returns a direct reference to the internal array. Modifying this array from any thread after construction will break the immutability contract of the class and lead to unpredictable behavior in the world generator. Callers must treat the returned array as read-only.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntries() | PrefabContainerEntry[] | O(1) | Returns the internal array of prefab rule entries. |
| getMaxSize() | int | O(1) | Returns the pre-calculated maximum size of any prefab across all entries. |
| **PrefabContainerEntry** | | | |
| getPrefabs() | IWeightedMap | O(1) | Retrieves the weighted map of available prefabs for this rule set. |
| getEnvironmentId() | int | O(1) | Returns the ID of the environment this entry applies to. |
| getExtents() | int | O(N) / O(1) | Calculates (on first call) and returns the largest dimension of any prefab in this entry. Subsequent calls are O(1). |
| getPrefabPatternGenerator() | PrefabPatternGenerator | O(1) | Retrieves the placement algorithm associated with this entry. |

## Integration Patterns

### Standard Usage

The PrefabContainer is consumed by feature placement services within the world generator. The service identifies the current environment, finds the matching entry in the container, and uses its contents to orchestrate prefab placement.

```java
// A world generator service retrieves the appropriate container
PrefabContainer prefabRules = worldConfig.getPrefabContainerForZone("forest");

// It iterates through the rules to find one for the current environment
for (PrefabContainer.PrefabContainerEntry entry : prefabRules.getEntries()) {
    if (entry.getEnvironmentId() == currentEnvironmentId) {
        // Use the pattern generator and prefabs to place a structure
        PrefabPatternGenerator generator = entry.getPrefabPatternGenerator();
        IWeightedMap<WorldGenPrefabSupplier> prefabs = entry.getPrefabs();
        
        // ... logic to select a prefab and generate placement points ...
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Modifying the Entry Array:** The most critical anti-pattern is modifying the array returned by `getEntries`. This violates the class's design contract and will have catastrophic, non-local effects on world generation.

    ```java
    // DO NOT DO THIS
    PrefabContainer.PrefabContainerEntry[] entries = container.getEntries();
    entries[0] = null; // This will cause NullPointerExceptions deep within the worldgen engine.
    ```

-   **Manual Instantiation in Game Logic:** Avoid constructing PrefabContainer objects manually within gameplay or feature placement logic. These objects represent static configuration and should be instantiated once from a definitive source (asset files) during server initialization.

## Data Pipeline

The PrefabContainer serves as a structured, in-memory representation of configuration data. It does not process streaming data but rather holds static data for processing by other systems.

> Flow:
> WorldGen JSON Asset -> Server Asset Loader -> **PrefabContainer** -> Biome/Feature Generator -> World Chunk Modification

