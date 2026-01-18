---
description: Architectural reference for BiomePatternGenerator
---

# BiomePatternGenerator

**Package:** com.hypixel.hytale.server.worldgen.biome
**Type:** Transient

## Definition
```java
// Signature
public class BiomePatternGenerator {
```

## Architecture & Concepts

The BiomePatternGenerator is a foundational component in the server-side world generation pipeline, responsible for deterministically selecting the correct biome for any given world coordinate. It operates as a stateful, configured service that encapsulates the logic for a specific region or "zone" of the world.

Its architecture is based on a two-layer procedural generation model:

1.  **Base Biome Layer:** This layer establishes the large-scale, contiguous biome regions like forests, deserts, and plains. It utilizes a point-based distribution algorithm, delegated to an **IPointGenerator** dependency. This approach is analogous to a Voronoi diagram, where the biome at any coordinate (x, z) is determined by the biome assigned to the closest "center point". This ensures that biome regions are cohesive and have well-defined, albeit complex, borders.

2.  **Custom Biome Layer:** This layer acts as a decorator or override system. After the base biome is determined, the generator iterates through a list of **CustomBiome** definitions. These are smaller, more specialized biomes, such as an oasis within a desert or a magical grove within a forest. Each CustomBiome has its own generator with specific placement rules, which are evaluated against the base biome and contextual data from the broader **ZoneGeneratorResult**. This allows for the creation of rich, nested environmental details without complicating the large-scale biome layout.

This generator is the source of truth for biome identity before the terrain itself is shaped, acting as a blueprint for subsequent generation stages like heightmap generation, block placement, and prefab spawning.

### Lifecycle & Ownership

-   **Creation:** A BiomePatternGenerator is instantiated by a higher-level orchestrator, typically a **ZoneGenerator**, at the beginning of a zone's generation process. It is configured with a specific point generator, a weighted map of tile biomes, and an array of custom biomes relevant only to that zone.
-   **Scope:** The object's lifetime is strictly bound to the generation of its parent zone. It is a short-lived, single-purpose service.
-   **Destruction:** It holds no persistent state and manages no external resources. It is eligible for garbage collection as soon as the zone generation task that created it completes.

## Internal State & Concurrency

-   **State:** The BiomePatternGenerator is stateful, but its state is **effectively immutable** after construction. The constructor accepts all necessary configuration (point generator, biome maps) and stores them in final fields. It also pre-computes and caches a combined list of all biomes and the maximum prefab extent (`extents`) for fast lookups during generation. No internal state is modified after initialization.

-   **Thread Safety:** This class is **thread-safe**. Due to its immutable-after-construction design, multiple worker threads can safely and concurrently call methods like `generateBiomeAt` without locks or synchronization. This is a critical architectural feature that enables the Hytale engine to parallelize world generation across multiple CPU cores, significantly reducing load times.

## API Surface

The public API provides the primary interface for querying the biome at a specific world coordinate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateBiomeAt(zoneResult, seed, x, z) | Biome | O(C) | The primary entry point. Determines the final biome by first resolving the base biome and then checking for any applicable custom biome overrides. C is the number of configured custom biomes. |
| getBiome(seed, x, z) | TileBiome | O(P) | Resolves the base TileBiome using the configured IPointGenerator. P is the complexity of the point generator's search algorithm. |
| getCustomBiomeAt(seed, x, z, zoneResult, parent) | CustomBiome | O(C) | Iterates through all custom biomes to find one that should override the given parent biome at the specified coordinates. |
| getExtents() | int | O(1) | Returns the pre-calculated maximum size of any prefab within the configured biomes. Used by other systems to determine necessary buffer regions. |
| getBiomes() | Biome[] | O(1) | Returns the cached, combined array of all TileBiomes and CustomBiomes. |

## Integration Patterns

### Standard Usage

The BiomePatternGenerator is not intended for direct instantiation by gameplay logic. It is created and used by a world generation orchestrator, which provides the necessary zone-specific configuration.

```java
// Example from a hypothetical ZoneGenerator
// 1. Configure the generator for this specific zone
IPointGenerator pointGenerator = zoneConfig.getPointGenerator();
IWeightedMap<TileBiome> tileBiomes = zoneConfig.getTileBiomes();
CustomBiome[] customBiomes = zoneConfig.getCustomBiomes();

BiomePatternGenerator biomeGenerator = new BiomePatternGenerator(pointGenerator, tileBiomes, customBiomes);

// 2. Use the generator during chunk generation
for (int x = 0; x < CHUNK_SIZE; x++) {
    for (int z = 0; z < CHUNK_SIZE; z++) {
        // The zoneResult provides broader context for rule evaluation
        Biome finalBiome = biomeGenerator.generateBiomeAt(zoneResult, worldSeed, worldX + x, worldZ + z);
        
        // ... proceed with terrain generation using finalBiome
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Global Caching:** Do not cache and reuse a BiomePatternGenerator across different zones. Each instance is tightly coupled to a specific zone's configuration. Reusing it would result in incorrect biome generation for other zones.
-   **State Modification:** Do not use reflection or other unsafe methods to modify the internal biome arrays after construction. This will break thread safety and lead to unpredictable, non-deterministic world generation bugs that are extremely difficult to debug.
-   **Direct Instantiation without Context:** Do not call `new BiomePatternGenerator(...)` with generic or placeholder data. The quality and character of the world depend entirely on the carefully configured biome maps and point generators provided by the zone system.

## Data Pipeline

The flow of data to determine a single biome is a multi-stage process encapsulated within the `generateBiomeAt` method.

> Flow:
> (Seed, X, Z, ZoneGeneratorResult) -> **BiomePatternGenerator.generateBiomeAt**
> 1.  Input coordinates are passed to the configured **IPointGenerator** to find the nearest biome center point.
> 2.  A deterministic hash of the center point's coordinates is used as a key to look up a base **TileBiome** from the **IWeightedMap**.
> 3.  The generator then iterates through its list of **CustomBiome** definitions.
> 4.  For each CustomBiome, its associated **CustomBiomeGenerator** evaluates a set of rules. These rules check if the base TileBiome is a valid parent and may use contextual data from the **ZoneGeneratorResult** (e.g., elevation, temperature).
> 5.  If a CustomBiome's rules pass, it is selected as the override. If multiple custom biomes could apply, the first one in the list wins.
> 6.  The final result, either the CustomBiome or the original TileBiome, is returned.
> -> **Final Biome** -> Subsequent WorldGen Stages (e.g., Terrain Carver, Prefab Placer)

