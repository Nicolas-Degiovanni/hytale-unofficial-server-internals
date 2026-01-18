---
description: Architectural reference for WaterContainer
---

# WaterContainer

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class WaterContainer {
```

## Architecture & Concepts
The WaterContainer is a specialized, immutable data structure that serves as a configuration model for procedural water generation. It is not a service or a manager; rather, it encapsulates a collection of rules that define where and at what height water bodies should be placed within the game world.

This class is a core component of the declarative world generation system. Instead of hard-coding water placement logic, developers define water properties in external assets (e.g., biome definition files). These assets are then deserialized into WaterContainer instances at runtime.

The primary architectural pattern is the **Composite Rule Evaluator**. The WaterContainer holds an array of WaterContainer.Entry objects. Each Entry represents a single, self-contained rule for a potential water body, complete with:
- A **placement condition** (the mask) to determine if the rule applies to a given coordinate.
- A **procedural height range** (min and max suppliers) to calculate the water level.

The `getMaxHeight` method acts as the evaluator, iterating over all composite rules (the Entries) and aggregating the results to determine the final, definitive water level for a specific world column. This design allows for complex, layered water features by simply adding more entries to the container.

### Lifecycle & Ownership
- **Creation:** WaterContainer instances are not created directly via their constructor in game logic. They are instantiated by the asset loading pipeline, typically during the deserialization of world generation configuration files (e.g., biome or zone definitions).
- **Scope:** The lifetime of a WaterContainer is tied to its owning configuration object. For instance, if it is part of a Biome definition, it will exist in memory as long as that Biome is loaded. It is a value object, not a persistent entity.
- **Destruction:** The object is managed by the Java Garbage Collector. No manual cleanup is required. It is reclaimed once the parent configuration object that references it goes out of scope.

## Internal State & Concurrency
- **State:** **Immutable**. The internal `entries` array is final and assigned only during construction. The nested WaterContainer.Entry class is also fully immutable. This guarantees that a WaterContainer's behavior is consistent and predictable throughout its lifetime.
- **Thread Safety:** **Fully thread-safe**. Due to its immutability, a WaterContainer instance can be safely shared and accessed by multiple world generation threads simultaneously without any need for locks or synchronization. The `getMaxHeight` method is a pure function, producing the same output for the same input without side effects. This is a critical design feature for enabling highly parallelized and performant world generation.

## API Surface
The public API is minimal, focusing exclusively on querying the aggregated water level.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMaxHeight(seed, x, z) | int | O(N) | Calculates the highest possible water level at a world coordinate by evaluating all contained entries. N is the number of entries. Returns NO_WATER_AT_COORDINATED if no entry produces a valid water level. |
| hasEntries() | boolean | O(1) | Returns true if the container holds one or more water generation rules. |
| getEntries() | WaterContainer.Entry[] | O(1) | Returns the internal array of rule entries. **Warning:** The returned array should be treated as read-only to preserve the immutability contract of the container. |

## Integration Patterns

### Standard Usage
The WaterContainer is intended to be used by higher-level world generation systems, such as a Biome or Chunk generator. The generator retrieves the container from its configuration and queries it for each column being processed.

```java
// Assume 'activeBiome' is the configuration object for the current region
WaterContainer waterRules = activeBiome.getWaterContainer();

if (waterRules.hasEntries()) {
    for (int x = 0; x < CHUNK_WIDTH; x++) {
        for (int z = 0; z < CHUNK_WIDTH; z++) {
            int worldX = chunk.getX() * CHUNK_WIDTH + x;
            int worldZ = chunk.getZ() * CHUNK_WIDTH + z;

            int waterHeight = waterRules.getMaxHeight(worldSeed, worldX, worldZ);

            if (WaterContainer.isValidWaterHeight(waterHeight)) {
                // ... logic to place water blocks up to waterHeight
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WaterContainer()`. These objects represent deserialized configuration. Manually creating them bypasses the asset pipeline and can lead to world generation that is inconsistent with its definitions.
- **State Modification:** Do not modify the array returned by `getEntries()`. While the array reference itself cannot be changed, its contents can be. Modifying an entry would violate the immutability guarantee and cause unpredictable behavior across different threads that may be sharing the same WaterContainer instance.
- **Per-Coordinate Caching:** Do not build external caches for the results of `getMaxHeight`. The underlying suppliers within each Entry may be complex noise functions. The calculation is designed to be fast enough to run for every required coordinate without memoization.

## Data Pipeline
The WaterContainer acts as a runtime representation of static configuration data. It is a crucial link between the asset definition and the procedural generation logic.

> Flow:
> Biome Definition Asset (JSON/HOCON) -> Asset Deserializer -> **WaterContainer Instance** -> Biome Generator -> `getMaxHeight(seed, x, z)` -> Final Water Level -> Chunk Voxel Data

