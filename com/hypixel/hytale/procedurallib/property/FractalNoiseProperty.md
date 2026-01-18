---
description: Architectural reference for FractalNoiseProperty
---

# FractalNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class FractalNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The FractalNoiseProperty class is a foundational component within the procedural generation library, responsible for defining and computing multi-layered, or fractal, noise. It does not implement a noise algorithm itself; rather, it orchestrates the repeated application of a base NoiseFunction over several iterations, known as octaves, to produce more complex and natural-looking patterns than a single noise function can achieve.

This class embodies the **Strategy Pattern**. It is configured with a specific `FractalFunction` (provided by the FractalMode enum) which dictates *how* the octaves are combined. Standard strategies like Fractional Brownian Motion (FBM), Billow, and Multi-Rigid are provided as pre-built implementations. FractalNoiseProperty acts as the context object, holding the chosen strategy along with essential parameters like the number of octaves, lacunarity (frequency change per octave), and persistence (amplitude change per octave).

This design effectively decouples the high-level configuration of fractal noise from the low-level noise primitives and combination algorithms. It allows world generation logic to treat fractal noise as a single, configurable property, greatly simplifying the creation of varied and complex procedural content.

### Lifecycle & Ownership
-   **Creation:** Instances are typically created by higher-level configuration loaders or world generation managers. They are intended to be instantiated once with a complete set of parameters that define a specific noise characteristic, such as continental elevation or cave system density.
-   **Scope:** This is a lightweight value object. Its lifetime is tied to the specific generation task that requires it. An instance may be cached as part of a larger world generation profile or created on-demand and subsequently garbage collected. It does not persist globally.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields are declared `final` and are initialized exclusively through the constructor. Once an instance is created, its configuration cannot be altered.
-   **Thread Safety:** **Fully thread-safe**. Its immutable nature guarantees that a single instance can be safely shared and accessed by multiple world-generation threads concurrently without any need for external synchronization or locks. This is a critical feature for enabling high-performance, parallelized chunk generation.

## API Surface
The public contract is focused on evaluating the configured noise at a given coordinate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(octaves) | Computes the 2D fractal noise value. Delegates to the configured FractalFunction strategy. |
| get(seed, x, y, z) | double | O(octaves) | Computes the 3D fractal noise value. Delegates to the configured FractalFunction strategy. |

## Integration Patterns

### Standard Usage
An instance should be configured once and reused for all evaluations related to a specific feature. The FractalMode enum is the standard source for the required FractalFunction strategy.

```java
// 1. Define a base noise function to be layered (e.g., Simplex)
NoiseFunction baseNoise = new SimplexNoise();

// 2. Configure the fractal properties using a standard FBM mode
FractalNoiseProperty terrainNoise = new FractalNoiseProperty(
    12345, // seedOffset
    baseNoise,
    FractalNoiseProperty.FractalMode.FBM.getFunction(),
    8,     // octaves
    2.0,   // lacunarity (frequency doubles each octave)
    0.5    // persistence (amplitude halves each octave)
);

// 3. Evaluate the noise at a specific coordinate for a given world seed
double worldSeed = 9876;
double value = terrainNoise.get(worldSeed, 150.5, 75.2);
```

### Anti-Patterns (Do NOT do this)
-   **Re-instantiation in Loops:** Never create a new FractalNoiseProperty inside a tight generation loop (e.g., for every block in a chunk). This is highly inefficient and generates significant GC pressure. Configure it once and reuse the instance.
-   **Parameter Misconfiguration:** Using invalid parameters will produce degenerate or useless output.
    -   **octaves < 2:** Using one octave defeats the purpose of fractal noise and is less performant than using the base NoiseFunction directly.
    -   **lacunarity <= 1.0:** Fails to increase the frequency, resulting in overlapping or non-detailed noise.
    -   **persistence >= 1.0:** Causes the amplitude to grow with each octave, leading to chaotic and uncontrolled output.
-   **Ignoring the FractalMode Enum:** While technically possible to implement the FractalFunction interface manually, it is strongly discouraged unless creating a novel fractal algorithm. The provided enum constants are optimized and tested.

## Data Pipeline
FractalNoiseProperty acts as a stateless processing node in the procedural generation pipeline. It takes coordinates and a seed as input and produces a single floating-point value as output based on its immutable internal configuration.

> Flow:
> (World Seed, Coordinates) -> **FractalNoiseProperty.get()** -> [Delegation to FractalFunction Strategy] -> [Internal Loop over *octaves*] -> Base NoiseFunction.get() -> [Value Aggregation per Strategy] -> Final Normalized Noise Value

