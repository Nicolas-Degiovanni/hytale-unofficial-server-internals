---
description: Architectural reference for ZoneBiomeResult
---

# ZoneBiomeResult

**Package:** com.hypixel.hytale.server.worldgen.chunk
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class ZoneBiomeResult {
```

## Architecture & Concepts
The ZoneBiomeResult class is a data structure, not a service. It functions as a value object or Data Transfer Object (DTO) specifically designed to operate within the server-side world generation pipeline.

Its primary role is to act as a data carrier, bundling the outputs from the zone and biome selection stages of world generation into a single, cohesive unit. For any given coordinate column in the world, this object encapsulates which zone it belongs to, the specific biome selected within that zone, and the raw noise values that influenced these decisions.

By aggregating this data, ZoneBiomeResult decouples the initial biome and height determination logic from subsequent terrain sculpting and block placement stages. This allows different parts of the world generator to operate on a consistent snapshot of data for a specific location without needing to re-query the initial noise maps or selection algorithms.

## Lifecycle & Ownership
- **Creation:** A new ZoneBiomeResult is instantiated on the fly by a high-level generator, such as a BiomeProvider or ChunkGenerator, for each coordinate column being processed. Its creation is an ephemeral event that occurs deep within the chunk generation process.

- **Scope:** Extremely short-lived. The object's lifetime is confined to the procedural generation of a single vertical column of blocks. It is created, populated, read by subsequent stages, and then immediately becomes eligible for garbage collection. It does not persist between ticks or across chunk boundaries.

- **Destruction:** The object is not explicitly destroyed. It is a lightweight, stack-allocated (or short-lived heap-allocated) object that is reclaimed by the Java garbage collector once the method scope in which it was created completes.

## Internal State & Concurrency
- **State:** Highly mutable. All internal fields are publicly accessible via standard getters and setters. The class is designed to be populated incrementally as the world generation pipeline progresses. It holds no internal caches or complex state; it *is* the state for a single point in the generation process.

- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** World generation is a heavily multi-threaded process. Each worker thread responsible for generating a chunk or a region must create and operate on its own distinct instances of ZoneBiomeResult. Sharing a single instance between threads will result in catastrophic race conditions, data corruption, and non-deterministic world generation.

## API Surface
The public API consists exclusively of standard accessors and mutators for its data fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ZoneBiomeResult() | constructor | O(1) | Creates an empty result object. |
| ZoneBiomeResult(...) | constructor | O(1) | Creates and fully populates the result object. |
| getZoneResult() | ZoneGeneratorResult | O(1) | Retrieves the result from the zone generation stage. |
| setZoneResult(...) | void | O(1) | Sets the result from the zone generation stage. |
| getBiome() | Biome | O(1) | Retrieves the final selected Biome for the coordinate. |
| setBiome(...) | void | O(1) | Sets the final selected Biome. |
| getHeightThresholdContext() | double | O(1) | Retrieves a noise value used for biome height transitions. |
| setHeightThresholdContext(...) | void | O(1) | Sets the biome height transition noise value. |
| getHeightmapNoise() | double | O(1) | Retrieves the primary noise value used for terrain height. |
| setHeightmapNoise(...) | void | O(1) | Sets the primary terrain height noise value. |

## Integration Patterns

### Standard Usage
This object is intended to be used as a temporary data container within a single, sequential block of world generation logic. The typical pattern is to create an instance, have various generator components populate it, and then pass the completed object to the final terrain composition stage.

```java
// Conceptual example within a world generator
ZoneGeneratorResult zoneData = zoneGenerator.getZoneFor(x, z);
Biome selectedBiome = biomeProvider.getBiomeFor(x, z, zoneData);
double heightNoise = noiseProvider.getHeightNoise(x, z);

// Create and populate the result DTO
ZoneBiomeResult result = new ZoneBiomeResult();
result.setZoneResult(zoneData);
result.setBiome(selectedBiome);
result.setHeightmapNoise(heightNoise);

// Pass the result to the next stage
terrainCarver.apply(chunk, result);
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching or Re-use:** Do not attempt to cache or re-use a ZoneBiomeResult instance for a different coordinate. This will cause data from one location to incorrectly influence the generation of another, leading to visual artifacts and chunk seams. Always create a new instance for each distinct coordinate column.
- **Cross-Thread Sharing:** Never pass an instance of this object from one world generation worker thread to another. This is a critical violation of its design and will lead to severe concurrency issues.
- **Long-Term Storage:** This object is not designed for serialization or long-term storage. It contains transient generation data, not final world state.

## Data Pipeline
ZoneBiomeResult serves as a critical data packet that flows between distinct stages of the world generation algorithm.

> Flow:
> Input Coordinates (X, Z) -> ZoneGenerator -> BiomeProvider -> **ZoneBiomeResult** -> TerrainHeightGenerator -> BlockPlacer -> Final Chunk Data

