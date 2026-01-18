---
description: Architectural reference for PrefabWeights
---

# PrefabWeights

**Package:** com.hypixel.hytale.server.core.prefab
**Type:** Data Structure / Model

## Definition
```java
// Signature
public class PrefabWeights {
```

## Architecture & Concepts

The PrefabWeights class is a specialized data structure designed to manage and execute weighted random selections. Its primary role within the server architecture is to facilitate procedural generation systems, such as biome population, structure placement, or loot table resolution, where different outcomes must occur with varying probabilities.

Conceptually, this class operates in two distinct phases: a **configuration phase** and a **runtime selection phase**.

1.  **Configuration:** An instance holds a map of string identifiers (prefab names) to their corresponding numeric weights. This mapping is typically defined in external data files and loaded into memory via the Hytale serialization framework, as indicated by the static **CODEC** field. This allows designers to tune probabilities without modifying source code.

2.  **Runtime Selection:** Upon the first selection request, the class performs a one-time, lazy initialization. It transforms the discrete weights map into a cumulative weight array. This array is a performance optimization that allows for fast, repeated selections by mapping a random value (from 0.0 to 1.0) to a specific index in the array.

The static **NONE** instance implements a Flyweight pattern, providing a shared, immutable, and memory-efficient representation for cases where no weights are defined, preventing the need for null checks in consuming systems.

## Lifecycle & Ownership

-   **Creation:** PrefabWeights instances are primarily created through deserialization from game data files using the provided static **CODEC**. They can also be instantiated programmatically via the public constructor or the static **parse** method for testing or dynamic generation scenarios.

-   **Scope:** The lifetime of a PrefabWeights object is bound to its owning configuration object. For example, if it defines monster spawn rates for a biome, it will remain in memory as long as the biome's configuration data is loaded.

-   **Destruction:** Instances are managed by the Java garbage collector and are eligible for cleanup once they are no longer referenced. There is no explicit destruction or resource cleanup method.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable during the configuration phase but becomes effectively immutable after the first selection call. The core state consists of the **weightsLookup** map. Upon the first call to the **get** method, the **sum** and **weights** array are calculated and cached.

    **WARNING:** Any modifications to the weights via **setWeight** or **removeWeight** after the first selection has occurred will **not** be reflected in subsequent selections. The cached cumulative weight array will become stale, leading to incorrect probabilistic behavior.

-   **Thread Safety:** This class is partially thread-safe. The lazy initialization logic is protected by a `synchronized` block and a `volatile` flag, employing a double-checked locking pattern. This ensures that the cumulative weight array is built safely and exactly once, even if multiple threads call **get** concurrently on an uninitialized instance.

    However, the class is **not** fully thread-safe. Methods that mutate the underlying weight map, such as **setWeight**, are not synchronized. Concurrent writes, or a write concurrent with a **get** call, will result in a race condition and undefined behavior. This class is designed for a write-once, read-many-times pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(T[] elements, Function nameFunc, Random random) | T | O(N) | Selects an element from the array based on configured weights. The first call has O(N) complexity to build the internal cache; subsequent calls are also O(N) due to the linear scan of the cumulative weights. |
| getWeight(String prefab) | double | O(1) | Retrieves the configured weight for a specific prefab name. |
| setWeight(String prefab, double weight) | void | O(1) | Sets the weight for a prefab. Throws IllegalArgumentException for negative weights. **WARNING:** Unsafe to call after the first selection. |
| parse(String mappingString) | PrefabWeights | O(M) | Static factory method to create an instance from a comma-delimited string. M is the length of the string. |

## Integration Patterns

### Standard Usage

The intended use is to load a PrefabWeights instance as part of a larger configuration object and use it for repeated weighted selections within a system like a world generator.

```java
// Assume 'zoneConfig' is an object deserialized from a data file
// and contains a 'structureWeights' field of type PrefabWeights.
PrefabWeights weights = zoneConfig.getStructureWeights();
StructurePrefab[] availableStructures = ...;
Random random = world.getRandom();

// Select a structure to place based on its configured weight
StructurePrefab chosenStructure = weights.get(availableStructures, StructurePrefab::getName, random);

if (chosenStructure != null) {
    world.placeStructure(chosenStructure);
}
```

### Anti-Patterns (Do NOT do this)

-   **Post-Initialization Mutation:** The most critical anti-pattern is modifying weights after the selection mechanism has been initialized. This leads to a desynchronized state.

    ```java
    // BAD: The second 'get' call uses stale cached data
    PrefabWeights weights = ...;
    weights.get(prefabs, ...); // Caches cumulative weights
    weights.setWeight("some_prefab", 500.0); // This change will be ignored
    weights.get(prefabs, ...); // Selection logic is now incorrect
    ```

-   **Re-using with Different Element Sets:** The internal cache is sized according to the first array of elements passed to **get**. Using the same instance with a different set of elements of a different size will cause runtime failures or incorrect selections.

    ```java
    // BAD: The cached 'weights' array has a length of 3
    PrefabWeights weights = ...;
    weights.get(new Prefab[3], ...);

    // This will fail the internal length check and return null
    weights.get(new Prefab[5], ...);
    ```

## Data Pipeline

PrefabWeights serves as a data container and runtime calculator within a larger data flow, typically originating from game configuration files.

> Flow:
> Game Data File (e.g., JSON) -> Hytale **Codec** System -> **PrefabWeights Instance** -> Procedural Generation Service -> **get()** -> Selected Game Object

