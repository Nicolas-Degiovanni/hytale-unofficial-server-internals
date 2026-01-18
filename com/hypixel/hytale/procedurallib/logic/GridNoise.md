---
description: Architectural reference for GridNoise
---

# GridNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Transient Utility

## Definition
```java
// Signature
public class GridNoise implements NoiseFunction {
```

## Architecture & Concepts
The **GridNoise** class is a specialized implementation of the **NoiseFunction** interface, designed to generate a deterministic, grid-like pattern. Architecturally, it serves as a foundational primitive within the procedural generation engine.

Unlike stochastic noise functions such as Perlin or Simplex noise which produce organic, pseudo-random patterns, **GridNoise** produces a perfectly regular, predictable output. Its primary role is to define geometric structures, boundaries, or cellular regions within a world. The output value represents the normalized distance to the nearest grid line on any axis, creating a pattern of intersecting "valleys" (values approaching -1.0) along integer coordinates and "plateaus" (values approaching 1.0) at the center of grid cells.

This component is frequently combined with other noise functions in a chain or composite structure. For example, it can be used as a mask to blend different biome noises or to carve perfectly straight roads or tunnels into otherwise chaotic terrain.

**WARNING:** The class name includes "Noise" but it does not produce random values. The `seed` and `offsetSeed` parameters required by the **NoiseFunction** interface are ignored entirely. Its output is solely dependent on the input coordinates and the thickness values provided at construction.

## Lifecycle & Ownership
- **Creation:** **GridNoise** is instantiated by higher-level procedural generation controllers, such as a **WorldGenerator** or a specific **BiomeFeatureGenerator**. It is configured with fixed thickness parameters that define the characteristics of the grid for a specific use case.
- **Scope:** The object is stateless and immutable. Its lifetime is typically tied to the configuration of the generator that created it. It may be a long-lived object held as a member of a biome definition or a short-lived object created on-the-fly for a specific generation task.
- **Destruction:** Managed entirely by the Java Garbage Collector. As it holds no external resources, no explicit cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are `final` and are initialized exclusively in the constructor. The object's state, defined by its thickness parameters, cannot be modified after creation.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature and lack of side effects, a single **GridNoise** instance can be safely shared and accessed by multiple worker threads simultaneously without any need for locks or synchronization. This is a critical property for enabling highly parallelized chunk and world generation.

## API Surface
The public contract is defined by the **NoiseFunction** interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, offsetSeed, x, y) | double | O(1) | Calculates the 2D grid value at the given coordinates. The seed parameters are ignored. |
| get(seed, offsetSeed, x, y, z) | double | O(1) | Calculates the 3D grid value at the given coordinates. The seed parameters are ignored. |

## Integration Patterns

### Standard Usage
**GridNoise** should be instantiated once with a specific configuration and then reused for all relevant calculations. It is typically used by a terrain sampler or feature placer that iterates over a volume of coordinates.

```java
// A generator configures a GridNoise instance for creating a "city block" pattern.
NoiseFunction gridPattern = new GridNoise(0.1, 0.1, 0.1);

// A terrain sampler later uses this instance to evaluate points in a chunk.
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        // The seed values 0, 0 are passed but have no effect.
        double value = gridPattern.get(0, 0, worldX + x, 0, worldZ + z);
        if (value < -0.8) {
            // This point is on a grid line, place a "road" block.
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instantiation in a Loop:** Never create a new **GridNoise** instance inside a tight generation loop. This is inefficient and creates unnecessary garbage collector pressure. Instantiate it once and reuse the instance.
- **Expecting Randomness:** Do not use this class expecting random or organic noise. It is a deterministic pattern generator. Using different seeds will have no effect on the output, which can be a source of significant confusion.
- **Ignoring the Output Range:** The output is normalized to the range [-1.0, 1.0]. Downstream consumers must be designed to handle this specific range. Assuming a [0, 1] range is a common error.

## Data Pipeline
The data flow for **GridNoise** is simple and direct, transforming spatial coordinates into a structural value.

> Flow:
> World Generator Configuration -> **GridNoise(thickness)** -> Terrain Sampler provides (x, y, z) -> **GridNoise.get()** -> double value [-1.0, 1.0] -> Block Placement Logic

