---
description: Architectural reference for TintContainer
---

# TintContainer

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Data Container

## Definition
```java
// Signature
public class TintContainer {
```

## Architecture & Concepts

The TintContainer is a data-driven, conditional rule engine responsible for determining the procedural color tint of world elements, such as foliage or water, at a specific world coordinate. It is a fundamental component of the world generation pipeline, enabling biome-specific and location-dependent color variations.

Architecturally, it functions as a prioritized list of rules. When queried for a color at a given (x, z) coordinate, it evaluates each rule in sequence. The first rule whose spatial condition is met is executed, and its result is returned. If no specific rule matches, a mandatory default rule is used as a fallback.

This system is composed of several key abstractions:
*   **TintContainerEntry:** Represents a single conditional rule. It binds a spatial condition to a color generation strategy.
*   **ICoordinateCondition:** A predicate that defines *where* a rule is active. This could be a check against a noise map, a specific altitude range, or other world generation data.
*   **NoiseProperty:** A procedural noise function (e.g., Perlin, Simplex) that provides a source of deterministic, pseudo-random values across the world coordinates. This value is the primary input for selecting a color.
*   **IWeightedMap:** A data structure that maps the continuous output from a NoiseProperty to a discrete set of integer color values. This is how the system translates a noise value into a final color from a predefined palette.

The entire structure is designed to be deserialized from configuration files, allowing designers to define complex biome coloring without modifying engine code.

### Lifecycle & Ownership
-   **Creation:** TintContainer instances are not intended for direct instantiation in game logic. They are created by the asset loading system, typically by deserializing a JSON or other data file that defines a biome's properties. This process occurs at server startup or during a hot-reload of world generation assets.
-   **Scope:** An instance of TintContainer has a static scope. Once loaded, it is treated as an immutable, shared resource. It persists for the entire server session and is reused for all world generation calculations that require it.
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down or when its parent configuration (e.g., a Biome definition) is unloaded or reloaded.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields, including the list of entries and the default entry, are final and are assigned exclusively during construction. The object's state cannot be modified after it has been created.
-   **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a single TintContainer instance can be safely accessed by multiple world generation threads simultaneously without any need for external locking or synchronization. This is a critical feature for enabling parallel chunk generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTintColorAt(seed, x, z) | int | O(N) | Calculates the final ARGB color value for the given world seed and coordinate. N is the number of conditional entries; this is typically a small constant. |

## Integration Patterns

### Standard Usage
The TintContainer is retrieved from a higher-level configuration object, such as a Biome definition. A world generator then invokes getTintColorAt during the block placement or chunk population phase to apply the correct color.

```java
// In a hypothetical WorldGenerator or BiomeProcessor
Biome targetBiome = world.getBiomeAt(x, z);
TintContainer foliageTint = targetBiome.getFoliageTintContainer();

// The world seed is essential for deterministic results
int worldSeed = world.getSeed();
int color = foliageTint.getTintColorAt(worldSeed, x, z);

// Apply the calculated color to the relevant block or entity
someBlock.setOverlayColor(color);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create a TintContainer using its constructor in gameplay code. Doing so bypasses the data-driven asset system and creates "magic" behavior that is difficult to debug and maintain. All tinting logic should be defined in world generation asset files.
-   **State Modification:** Do not attempt to modify the internal list of entries via reflection or other means. The engine's performance and stability rely on the guaranteed immutability of this object for safe multi-threaded access.
-   **Ignoring the Seed:** Failing to pass the correct world seed will result in non-deterministic and inconsistent color generation, producing visible seams between chunks generated at different times.

## Data Pipeline
The flow of data to calculate a single color value is a multi-stage process orchestrated by the TintContainer.

> Flow:
> (seed, x, z) -> **TintContainer.getTintColorAt** -> [Iterate TintContainerEntry list] -> Entry.shouldGenerate(using ICoordinateCondition) -> [If true, select entry] -> Entry.getTintColorAt -> NoiseProperty.get -> (float noiseValue) -> IWeightedMap.get(noiseValue) -> **Final Color (int)**

