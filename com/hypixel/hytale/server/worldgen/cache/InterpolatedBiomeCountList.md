---
description: Architectural reference for InterpolatedBiomeCountList
---

# InterpolatedBiomeCountList

**Package:** com.hypixel.hytale.server.worldgen.cache
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class InterpolatedBiomeCountList {
```

## Architecture & Concepts
The InterpolatedBiomeCountList is a specialized, short-lived data accumulator used by the server's world generation pipeline. Its primary function is to collect and aggregate biome data from a central point and its immediate neighbors. This process is fundamental to the biome interpolation system, which is responsible for creating smooth, natural-looking transitions between different biomes.

This class acts as a temporary cache during the calculation for a single world coordinate. It gathers raw ZoneBiomeResult objects, filters them based on their distance from a central biome, and averages key properties like heightmapNoise. The resulting aggregated data is then consumed by subsequent world generation stages to calculate final terrain height, foliage density, and other biome-specific features. By averaging these values, the generator avoids sharp, unrealistic cliffs or seams where biomes meet.

It is not a general-purpose collection but a highly optimized component designed for a specific, performance-critical task within the world generator.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its default constructor, typically at the beginning of a localized terrain generation task. The world generation orchestrator is the owner.
- **Scope:** The object's lifetime is extremely brief and is scoped to the calculation of a single point or column in the world. It is created, populated, read, and then immediately discarded.
- **Destruction:** The object becomes eligible for garbage collection as soon as the method that created it completes. There are no external references or cleanup procedures required.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its internal state, including the collection of biome results and the central biome reference, is built up through successive calls to setCenter and add. The object is effectively a stateful builder.

- **Thread Safety:** This class is **not thread-safe**. The underlying fastutil collections are unsynchronized. Concurrent modification from multiple threads will result in a corrupted state and unpredictable behavior.

    **Warning:** This object must only be used within the context of a single world-generation thread. It is designed for serial, single-threaded access as part of a larger, parallelized chunk generation system where each worker thread manages its own instances.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setCenter(ZoneBiomeResult result) | void | O(1) | Initializes the collection. Establishes the central biome against which all subsequent additions are measured. **Must be called first.** |
| add(ZoneBiomeResult result, int distance2) | void | O(1) | Adds a neighboring biome's data. The data is only added if it falls within the interpolation radius of the center biome. If the biome already exists, its properties are averaged. |
| get(Biome biome) | BiomeCountResult | O(1) | Retrieves the aggregated result for a specific biome. |
| get(int index) | BiomeCountResult | O(1) | Retrieves the aggregated result for a specific biome ID. |
| getBiomeIds() | IntList | O(1) | Returns the list of unique biome IDs that have been collected. |

## Integration Patterns

### Standard Usage
The class follows a strict create-populate-use-discard pattern. The caller is responsible for orchestrating the population of data before consuming the results.

```java
// How a developer should normally use this
// 1. Create a new instance for a specific world coordinate calculation.
InterpolatedBiomeCountList biomeList = new InterpolatedBiomeCountList();

// 2. Set the center biome using the result from the primary coordinate.
ZoneBiomeResult centerResult = worldGen.getBiomeAt(x, z);
biomeList.setCenter(centerResult);

// 3. Iterate over neighbors and add them to the list.
for (Neighbor neighbor : getNeighbors(x, z)) {
    ZoneBiomeResult neighborResult = worldGen.getBiomeAt(neighbor.x, neighbor.z);
    int distanceSq = calculateDistanceSquared(x, z, neighbor.x, neighbor.z);
    biomeList.add(neighborResult, distanceSq);
}

// 4. Use the aggregated data for final terrain generation.
IntList biomeIds = biomeList.getBiomeIds();
// ... process results ...

// 5. Discard the instance. It is no longer needed.
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse an instance of this class for a new world coordinate calculation. Its internal state is not cleared, which will lead to incorrect data blending from previous calculations. Always create a new instance.
- **Incorrect Initialization:** Calling add before setCenter will result in a NullPointerException, as the central biome reference is null. The initialization order is critical.
- **Concurrent Access:** Do not share an instance across multiple threads. This will lead to race conditions and data corruption within the internal maps and lists.

## Data Pipeline
This class sits in the middle of the biome generation pipeline, transforming raw, point-in-time biome data into a smoothed, localized aggregate.

> Flow:
> World Generator -> Raw ZoneBiomeResult -> **InterpolatedBiomeCountList** (Filter & Average) -> Aggregated Biome Data -> Final Terrain Heightmap Generator

