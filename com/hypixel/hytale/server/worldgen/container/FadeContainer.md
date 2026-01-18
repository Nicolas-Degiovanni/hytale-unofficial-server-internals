---
description: Architectural reference for FadeContainer
---

# FadeContainer

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Transient

## Definition
```java
// Signature
public class FadeContainer {
```

## Architecture & Concepts

The FadeContainer is an immutable data-holding class that defines the rules for blending one world generation zone into another. It is a fundamental component of the procedural world generation pipeline, specifically responsible for calculating the smooth transition gradient between adjacent zones.

Its primary role is to translate a point's distance from a zone border into a normalized blending factor, a value between 0.0 and 1.0. This factor is then consumed by higher-level world generation services to mix terrain heights, biome properties, or other generated features.

The class distinguishes between two types of transitions, controlled by separate parameters:
*   **Mask Fading:** Governs the blending of abstract data layers, such as biome masks or influence maps.
*   **Terrain Fading:** Governs the blending of the final terrain heightmap, ensuring no sharp cliffs or discontinuities appear at zone boundaries.

The constant NO_FADE_HEIGHTMAP serves as a sentinel value to disable heightmap-based fading entirely for a given zone transition.

## Lifecycle & Ownership

-   **Creation:** FadeContainer instances are typically instantiated by a zone configuration loader or a parent ZoneGenerator service. The parameters are derived from data files (e.g., JSON definitions) that describe the properties of a worldgen zone and its relationship to its neighbors.
-   **Scope:** An instance of FadeContainer is short-lived and task-scoped. It is created to support a specific world generation operation for a chunk or region and is eligible for garbage collection once that operation is complete. It does not persist.
-   **Destruction:** Managed entirely by the Java garbage collector. No explicit cleanup is required due to its simple, state-less nature.

## Internal State & Concurrency

-   **State:** **Immutable**. All internal fields are declared as final and are initialized exclusively within the constructor. Once a FadeContainer is created, its configuration cannot be changed.

-   **Thread Safety:** **Fully thread-safe**. Its immutable design guarantees that it can be safely accessed and shared across multiple world generation threads without any need for locks, synchronization, or other concurrency controls. This is a critical property for enabling a high-performance, parallelized world generator.

## API Surface

The public API is designed for calculating blending factors based on world-space information. Simple getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMaskFactor(result) | double | O(1) | Calculates the blending factor for biome masks based on the distance from the zone border provided in the ZoneGeneratorResult. |
| getTerrainFactor(result) | double | O(1) | Calculates the blending factor for the terrain heightmap based on the distance from the zone border. |
| shouldFade() | boolean | O(1) | Returns true if the container is configured to perform heightmap-based fading. Critical check to avoid unnecessary calculations. |

## Integration Patterns

### Standard Usage

The FadeContainer is not a service to be retrieved from a context. It is a data object passed into the core world generation algorithms. A generator will use it to determine the weight of a zone's contribution to a final block or column.

```java
// A worldgen algorithm receives a FadeContainer, likely from a Zone definition
void processColumn(int x, int z, ZoneGeneratorResult result, FadeContainer fade) {
    // Calculate how much this zone's terrain should influence the final height
    double terrainInfluence = fade.getTerrainFactor(result);

    // If this value is < 1.0, the algorithm must sample the neighboring zone
    // and blend the results.
    if (terrainInfluence < 1.0) {
        Heightmap thisZoneHeight = calculateHeightFor(x, z);
        Heightmap neighborZoneHeight = getNeighborZoneHeight(x, z);
        
        // Blend the two heightmaps using the calculated factor
        Heightmap finalHeight = blend(thisZoneHeight, neighborZoneHeight, terrainInfluence);
    }
    // ...
}
```

### Anti-Patterns (Do NOT do this)

-   **Hardcoding Values:** Do not instantiate FadeContainer with magic numbers directly in the generation code. These parameters are tuning-sensitive and must be loaded from external zone configuration files to allow for data-driven design.
-   **Re-implementing Logic:** Do not attempt to read the fade parameters and re-implement the blending calculation. The `get...Factor` methods encapsulate the specific curve (linear gradient) and clamping logic and should be treated as the single source of truth.

## Data Pipeline

The FadeContainer is a pure computation component. It takes positional data as input and produces a blending factor as output.

> Flow:
> ZoneGeneratorResult (containing distance from border) -> **FadeContainer**.getFactor() -> Blending Factor (double) -> Terrain & Biome Mixer -> Final World Data

