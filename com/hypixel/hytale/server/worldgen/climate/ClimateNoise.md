---
description: Architectural reference for ClimateNoise
---

# ClimateNoise

**Package:** com.hypixel.hytale.server.worldgen.climate
**Type:** Transient

## Definition
```java
// Signature
public class ClimateNoise {
```

## Architecture & Concepts

The ClimateNoise class is a high-level orchestrator within the server-side world generation pipeline. Its primary responsibility is to synthesize multiple layers of procedural noise and cellular grid data into a definitive climate value for any given world coordinate. It acts as the nexus where abstract mathematical noise is translated into tangible geographical and biome-defining properties.

This component does not generate noise itself. Instead, it is configured with several independent NoiseProperty instances—typically for continentality, temperature, and humidity/intensity—and a ClimateGraph which defines the relationships between them.

The core architectural concept is **layered and distorted noise sampling**.
1.  A base **continent** noise value is sampled at the true world coordinate to determine the fundamental landmass or ocean distribution.
2.  A separate cellular **grid** function is evaluated to produce a distorted coordinate. This is a critical step that breaks up the uniform, parallel nature of standard Perlin or Simplex noise, leading to more organic and varied climate zone shapes.
3.  **Temperature** and **intensity** noise values are then sampled at this *distorted* coordinate.
4.  These final values are used to query a ClimateGraph, which acts as a lookup table to find the appropriate climate ID.
5.  Finally, the continent value is processed against a set of thresholds to produce bit flags identifying features like oceans, shorelines, and islands. The final output is a single integer packing both the climate ID and these feature flags.

## Lifecycle & Ownership

-   **Creation:** A ClimateNoise instance is created during the server's world generation setup phase. It is constructed with all its dependencies (Grid, NoiseProperty objects, Thresholds), which are typically loaded from world generation configuration files. It is not intended for on-the-fly instantiation during generation.
-   **Scope:** The object is designed to be long-lived, persisting for the entire world generation session for a given dimension or world. Its internal state is immutable, making it safe to reuse across the entire world map.
-   **Destruction:** The object holds no native resources and requires no explicit cleanup. It is garbage collected when the parent world generator is discarded.

## Internal State & Concurrency

-   **State:** The ClimateNoise object is **immutable**. All of its primary fields are declared final and are set only once at construction. This immutability is a cornerstone of its design, ensuring predictable and repeatable output. The nested Grid and Thresholds classes follow the same immutable pattern.

-   **Thread Safety:** This class is **thread-safe and re-entrant**. The generate method operates exclusively on its immutable internal state and the arguments provided by the caller.

    **WARNING:** While the ClimateNoise object itself is thread-safe, the required ClimateNoise.Buffer argument is a mutable state container. Callers in a multi-threaded environment **must** provide a separate Buffer instance for each worker thread to prevent data corruption and race conditions. Using a single shared buffer is a critical anti-pattern.

## API Surface

The public API is minimal, centered on the primary generation function.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(seed, x, y, buffer, climate) | int | O(1) | The primary generation function. Calculates the final climate integer for a coordinate, populating the provided buffer with intermediate results. |

## Integration Patterns

### Standard Usage

A world generator service should instantiate ClimateNoise once from configuration. For parallel chunk generation, a thread-local pool of ClimateNoise.Buffer objects should be maintained.

```java
// In a WorldGenerator initialization phase
ClimateNoise.Grid grid = ... // from config
NoiseProperty continentNoise = ... // from config
// ... other properties
ClimateNoise climateSystem = new ClimateNoise(grid, continentNoise, ...);

// In a per-thread chunk generation task
// Retrieve a buffer unique to this thread
ClimateNoise.Buffer threadLocalBuffer = bufferPool.get();
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        int worldX = chunkX * 16 + x;
        int worldZ = chunkZ * 16 + z;
        int climateData = climateSystem.generate(worldSeed, worldX, worldZ, threadLocalBuffer, climateGraph);
        // Use climateData and threadLocalBuffer.fade, .temperature, etc.
    }
}
// Return buffer to pool
bufferPool.release(threadLocalBuffer);
```

### Anti-Patterns (Do NOT do this)

-   **Instantiation in a Loop:** Never create a new ClimateNoise object inside a generation loop. It is a heavyweight object designed for reuse.
-   **Shared Buffer:** Do not share a single ClimateNoise.Buffer instance across multiple threads. This will cause severe data corruption as threads overwrite each other's intermediate results.
-   **Ignoring Buffer Output:** The generate method populates the buffer with valuable data like continentality, temperature, and biome fade values. Ignoring this data and only using the returned integer may result in missed opportunities for more nuanced features, such as smooth biome transitions or heightmap modifications based on raw noise values.

## Data Pipeline

The flow of data through the generate method is a multi-stage pipeline that transforms a simple coordinate into a complex, data-rich climate value.

> Flow:
> 1.  Input (worldX, worldY, seed)
> 2.  **Grid.eval** -> Distorts (worldX, worldY) into (distortedX, distortedY)
> 3.  **NoiseProperty.get(continent)** -> Samples continent noise at (worldX, worldY)
> 4.  **NoiseProperty.get(temperature, intensity)** -> Samples climate noise at (distortedX, distortedY)
> 5.  **ClimateGraph.indexOf(temperature, intensity)** -> Looks up a climate ID and fade value
> 6.  **getContinentFlags(continent)** -> Compares continent value against Thresholds to produce feature flags (ocean, shore, island)
> 7.  **Bitwise OR** -> Combines the climate ID and feature flags into a single integer
> 8.  Output: Packed integer and populated Buffer object

