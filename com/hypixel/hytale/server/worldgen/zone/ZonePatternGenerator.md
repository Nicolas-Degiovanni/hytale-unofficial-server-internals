---
description: Architectural reference for ZonePatternGenerator
---

# ZonePatternGenerator

**Package:** com.hypixel.hytale.server.worldgen.zone
**Type:** Transient

## Definition
```java
// Signature
public class ZonePatternGenerator {
```

## Architecture & Concepts

The ZonePatternGenerator is a core component of the server-side procedural world generation engine. Its primary responsibility is to deterministically select a specific Zone for any given world coordinate (x, z). It acts as a high-level procedural function that translates a coordinate into a categorical biome or area type.

This class implements a sophisticated, multi-layered generation strategy to create complex and natural-looking zone layouts:

1.  **Macro-Level Masking:** It first consults a **MaskProvider**. This provider acts as a large-scale "continent map" or control texture. It takes the world coordinates, applies a potential distortion, and returns an integer mask value. This mask dictates the *set* of possible zones for a large region.

2.  **Categorical Mapping:** The integer mask is then used as a key into the **ZoneColorMapping**. This lookup table translates the abstract mask value into a concrete array of candidate Zone objects. For example, a "green" mask value might map to an array containing {Forest, Plains, Swamp}.

3.  **Micro-Level Point Generation:** If the mapping returns an array with a single Zone, the decision is made. However, if multiple candidate Zones exist, the **IPointGenerator** is invoked. This is typically a Voronoi or Worley noise generator which creates a grid of feature points. The generator finds the nearest point to the input coordinate, and a hash of that point's location is used to deterministically select one Zone from the candidate array. This process carves the larger masked region into smaller, irregular sub-zones, creating the final pattern.

This two-tiered approach allows for both broad, artist-directed control (via the mask) and fine-grained, procedural detail (via the point generator).

### Lifecycle & Ownership
-   **Creation:** An instance is created via its public constructor, typically by a higher-level world generation orchestrator during the initialization of a world or dimension. It is configured with a specific set of dependencies (MaskProvider, Zone arrays, etc.) that define the generation rules for that world.
-   **Scope:** The object's lifetime is tied to a specific world generation session. It is not a global singleton and holds configuration data that may be unique to one world.
-   **Destruction:** The object contains no unmanaged resources and is subject to standard garbage collection once the world generator that created it is finished and releases its reference.

## Internal State & Concurrency
-   **State:** The ZonePatternGenerator is **immutable**. All of its dependency fields are marked final and are set only during construction. This design is critical for ensuring that world generation is perfectly reproducible given the same seed and configuration.
-   **Thread Safety:** The class itself contains no locks and manages no mutable state. It is therefore **conditionally thread-safe**. An instance can be safely shared and used across multiple chunk generation threads, provided that its injected dependencies (IPointGenerator, MaskProvider) are also thread-safe. This is a common and necessary pattern for a high-performance, multi-threaded world generation engine.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(seed, x, z) | ZoneGeneratorResult | O(1) | Primary entry point. Determines the Zone for a coordinate. **Warning:** This overload allocates a new ZoneGeneratorResult on every call and can cause GC pressure in tight loops. |
| generate(seed, x, z, result) | ZoneGeneratorResult | O(1) | A performance-oriented variant that populates a pre-allocated result object, avoiding heap allocation. This is the preferred method for batch processing. |
| getZones() | Zone[] | O(1) | Returns the complete list of zones this generator is configured to work with. |
| getUniqueZones() | Zone.Unique[] | O(1) | Returns the list of unique zones this generator is configured to work with. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve a pre-configured instance from a central world generation context and use it to determine the zone for a block column or sample point. For performance, a single ZoneGeneratorResult object should be reused for multiple calls.

```java
// Assume 'worldGenContext' provides the configured generator
ZonePatternGenerator generator = worldGenContext.getZonePatternGenerator();
ZoneGeneratorResult result = new ZoneGeneratorResult(); // Allocate once

for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        // Re-use the result object to avoid allocations
        generator.generate(worldSeed, chunkX * 16 + x, chunkZ * 16 + z, result);
        
        // Process the result...
        Zone currentZone = result.getZone();
        double borderDist = result.getBorderDistance();
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ZonePatternGenerator()`. The constructor requires a complex set of configured dependencies (MaskProvider, ZoneColorMapping, etc.) that define the world's structure. Manually constructing this object will result in a generator that is out of sync with the server's world definition. Always retrieve it from the appropriate context.
-   **Loop Allocation:** Avoid calling the `generate(seed, x, z)` overload inside performance-critical loops, such as during chunk generation. The repeated allocation of ZoneGeneratorResult objects will create significant garbage collector pressure and degrade server performance.

## Data Pipeline
The flow of data through the `generate` method is a multi-stage procedural evaluation.

> Flow:
> (seed, x, z) -> **MaskProvider** -> (mask_value, distorted_x, distorted_z) -> **ZoneColorMapping** -> Zone[] candidates -> **Conditional Logic**
>
> *   **If** candidates.length == 1 -> Final Zone Selected
> *   **If** candidates.length > 1 -> (seed, x, z) -> **IPointGenerator** -> nearest_point -> **HashUtil** -> index -> Final Zone Selected
>
> -> **ZoneGeneratorResult** (Contains Final Zone and Border Distance)

