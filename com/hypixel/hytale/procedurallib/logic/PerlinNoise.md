---
description: Architectural reference for PerlinNoise
---

# PerlinNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Transient

## Definition
```java
// Signature
public class PerlinNoise implements NoiseFunction {
```

## Architecture & Concepts
The PerlinNoise class is a concrete implementation of the NoiseFunction interface, providing a foundational algorithm for procedural content generation. It serves as a source of classic gradient noise, which is essential for creating natural-looking, pseudo-random patterns used in terrain heightmaps, biome distribution, texture generation, and other visual effects.

Architecturally, this class is a self-contained, mathematical primitive within the procedural library. It operates as a pure function, transforming spatial coordinates (x, y, z) and seed values into a continuous noise value. Its primary dependency is the GeneralNoise utility class, which provides the low-level mathematical building blocks for gradient vector calculation and interpolation.

The key design feature of this implementation is its configurable smoothness, achieved by injecting a GeneralNoise.InterpolationFunction at construction. This decouples the core Perlin noise algorithm from the specific smoothing curve used, allowing developers to select different interpolation strategies (e.g., linear, cosine, quintic) to control the final appearance of the noise. This is an application of the Strategy design pattern.

## Lifecycle & Ownership
-   **Creation:** An instance is created directly via its public constructor: `new PerlinNoise(interpolationFunction)`. It is expected to be instantiated by higher-level systems responsible for orchestrating world generation, such as a BiomeGenerator or a TerrainPass controller.
-   **Scope:** The object is transient and typically short-lived. Its scope is confined to the specific generation task for which it was created. For example, an instance might exist only for the duration of a single chunk's terrain generation pass.
-   **Destruction:** The object contains no unmanaged resources and requires no explicit cleanup. It is marked for garbage collection once the orchestrating system that created it releases its reference.

## Internal State & Concurrency
-   **State:** The PerlinNoise class is **immutable**. Its only internal state is the `interpolationFunction` field, which is marked as `final` and set exclusively during construction. It does not cache any computed values.
-   **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature and lack of side effects, the `get` methods can be safely invoked by multiple threads concurrently without any external synchronization. This is a critical property for modern procedural generation engines that heavily parallelize tasks like chunk generation.

## API Surface
The public contract is defined by the NoiseFunction interface, providing methods to sample noise in 2D and 3D space.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, offsetSeed, x, y) | double | O(1) | Computes the 2D Perlin noise value for the given coordinates. |
| get(seed, offsetSeed, x, y, z) | double | O(1) | Computes the 3D Perlin noise value for the given coordinates. |
| getInterpolationFunction() | InterpolationFunction | O(1) | Returns the interpolation function configured at construction. |

## Integration Patterns

### Standard Usage
The class is intended to be instantiated once per configured noise layer and reused for all coordinate lookups within that layer. The choice of interpolation function is a critical parameter that dictates the visual quality of the output.

```java
// A higher-level system configures the desired noise behavior.
// QUINTIC provides a high-quality, smooth curve.
GeneralNoise.InterpolationFunction smoothCurve = GeneralNoise.InterpolationFunction.QUINTIC;
NoiseFunction terrainNoise = new PerlinNoise(smoothCurve);

// The function is then used repeatedly to build a heightmap.
int worldSeed = 12345;
for (int x = 0; x < 256; x++) {
    for (int z = 0; z < 256; z++) {
        // Scale coordinates to control noise frequency.
        double sampleX = x / 64.0;
        double sampleZ = z / 64.0;
        double heightValue = terrainNoise.get(worldSeed, 0, sampleX, sampleZ);
        // Use heightValue to determine terrain elevation...
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Re-instantiation in Loops:** Never create a new PerlinNoise instance inside a tight loop that samples many points. This is extremely wasteful and will severely degrade generation performance. Instantiate it once and reuse the object.
    ```java
    // BAD: Creates thousands of unnecessary objects.
    for (int x = 0; x < 256; x++) {
        NoiseFunction badNoise = new PerlinNoise(GeneralNoise.InterpolationFunction.QUINTIC);
        double value = badNoise.get(seed, 0, x, 0);
    }
    ```
-   **Ignoring Interpolation Choice:** The default Java enum order is not a valid selection criteria. The choice between LINEAR, COSINE, and QUINTIC has a significant impact on visual quality. Using LINEAR can result in noticeable grid-aligned artifacts.

## Data Pipeline
PerlinNoise acts as a data source in a procedural generation pipeline. It does not transform incoming data; rather, it generates new data based on coordinate inputs.

> Flow:
> Procedural Generator -> Provides (Seed, X, Y, Z) coordinates -> **PerlinNoise** -> Calculates gradient vectors and performs interpolation -> Returns a double value between -1.0 and 1.0 -> Procedural Generator -> Maps the value to a game parameter (e.g., terrain height, temperature, block type).

