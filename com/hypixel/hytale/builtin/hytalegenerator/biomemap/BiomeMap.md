---
description: Architectural reference for BiomeMap
---

# BiomeMap

**Package:** com.hypixel.hytale.builtin.hytalegenerator.biomemap
**Type:** Framework Component

## Definition
```java
// Signature
public abstract class BiomeMap<V> extends BiCarta<BiomeType> {
```

## Architecture & Concepts
The BiomeMap class is an abstract base class that serves as a foundational data structure within the Hytale world generation pipeline. It represents a two-dimensional grid where each cell contains a BiomeType, effectively defining the biome layout for a specific region of the world.

By extending BiCarta, BiomeMap establishes a specialized contract for cartographic data focused exclusively on biomes. It acts as an intermediary data container, holding the output of biome placement algorithms before that data is consumed by subsequent world-building stages, such as terrain heightmap generation, vegetation placement, and structure spawning. Its primary role is to decouple the logic of *determining* biomes from the logic of *rendering* them into the game world.

## Lifecycle & Ownership
- **Creation:** A concrete implementation of BiomeMap is instantiated by a high-level world generator service (e.g., a ZoneGenerator) at the beginning of a regional generation task. It is never created directly by game logic.
- **Scope:** The lifetime of a BiomeMap instance is transient and strictly scoped to a single, self-contained world generation operation. It exists only as long as needed to build a specific world chunk or zone.
- **Destruction:** The object is eligible for garbage collection as soon as the world generation pipeline has consumed its data to produce the final world state for the region. There is no persistent reference to the BiomeMap after the generation task completes.

## Internal State & Concurrency
- **State:** This abstract class defines no state. Concrete subclasses are expected to be highly mutable, containing a 2D array or similar data structure to hold BiomeType values. The state represents a snapshot of the biome layout at a specific point in the generation process.
- **Thread Safety:** **Not thread-safe.** A BiomeMap instance is designed to be owned and modified by a single worker thread during a world generation task. Concurrent writes from multiple threads will result in data corruption and non-deterministic world generation. Any multi-threaded access must be managed by an external synchronization mechanism, typically by partitioning work so that threads operate on separate, independent BiomeMap instances.

## API Surface
The public API is inherited from its parent, BiCarta. The contract is expected to provide standard 2D grid manipulation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int x, int y) | BiomeType | O(1) | (Inherited) Retrieves the biome at the specified coordinates. |
| set(int x, int y, BiomeType) | void | O(1) | (Inherited) Assigns a biome to the specified coordinates. |

**WARNING:** The specific generic parameter V in the class signature is not used in the provided snippet. Its purpose is likely defined in concrete implementations or the parent BiCarta class, potentially for associating auxiliary metadata with the map.

## Integration Patterns

### Standard Usage
A BiomeMap is used as a data carrier between distinct stages of the world generator. A biome placement strategy populates the map, and subsequent terrain and feature generators read from it.

```java
// A generator service obtains a concrete BiomeMap instance
BiomeMapImplementation biomeLayout = generatorContext.createBiomeMap(region);

// A biome placement algorithm populates the map based on noise or other inputs
BiomePlacementStrategy.populate(biomeLayout, worldSeed, region.getBounds());

// Downstream systems consume the data
TerrainHeightGenerator.apply(worldData, biomeLayout);
FoliagePlacer.apply(worldData, biomeLayout);
```

### Anti-Patterns (Do NOT do this)
- **Shared State:** Do not pass the same BiomeMap instance to multiple concurrent generation threads for modification. This will cause severe race conditions. Each thread should operate on its own distinct map for the region it is responsible for.
- **Long-Term Storage:** Do not hold references to BiomeMap instances after the world generation task is complete. They are intermediate, transient objects and are not designed for persistence.

## Data Pipeline
The BiomeMap is a critical component in the procedural generation data flow, acting as a structured container for biome classification data.

> Flow:
> World Seed -> Noise Functions -> Biome Placement Algorithm -> **BiomeMap** -> Terrain Generator -> Final World Data

