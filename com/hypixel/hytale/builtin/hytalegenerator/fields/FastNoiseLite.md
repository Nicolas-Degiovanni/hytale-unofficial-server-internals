---
description: Architectural reference for FastNoiseLite
---

# FastNoiseLite

**Package:** com.hypixel.hytale.builtin.hytalegenerator.fields
**Type:** Stateful Utility

## Definition
```java
// Signature
public class FastNoiseLite {
```

## Architecture & Concepts
The **FastNoiseLite** class is a high-performance, self-contained procedural noise generation library. It serves as a foundational component within the world generation system, responsible for creating a wide variety of deterministic, pseudo-random patterns used to define terrain, biome distribution, cave systems, and other natural features.

Architecturally, this class is a pure computational engine. It does not integrate with other engine systems like networking, rendering, or events. Instead, it functions as a low-level tool invoked by higher-level generator services. Its design is centered on a "configure-then-use" pattern:
1.  An instance of **FastNoiseLite** is created.
2.  Its behavior is configured through a comprehensive set of public setter methods (**setSeed**, **setFrequency**, **setNoiseType**, **setFractalType**, etc.). These parameters collectively define a unique, infinite noise field.
3.  The **getNoise** methods are then called with 2D or 3D coordinates to sample a value from this configured noise field.

The class encapsulates multiple industry-standard noise algorithms, including OpenSimplex2, Perlin, and Cellular noise. It can further combine these base patterns into more complex ones using fractal algorithms (**FBm**, **Ridged**) and distort the input coordinate space via Domain Warping to produce more organic and visually interesting results.

## Lifecycle & Ownership
- **Creation:** An instance is created directly using its constructor, for example, `new FastNoiseLite(seed)`. It is a Plain Old Java Object (POJO) and is not managed by a dependency injection framework or service locator. Ownership is explicitly held by the system that instantiates it, typically a world generator or a specific feature generator.

- **Scope:** The lifetime of a **FastNoiseLite** object is tied to its owner. It may be short-lived, existing only for the duration of a single chunk generation task, or it may persist for the entire server session if it represents a core part of the world's generation rules.

- **Destruction:** The object is eligible for garbage collection once all references to it are released. It holds no native resources and does not require an explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The **FastNoiseLite** class is highly **mutable**. Its internal state consists of numerous configuration fields (**mSeed**, **mFrequency**, **mNoiseType**, etc.) that are modified via public setters. The output of the **getNoise** methods is a pure function of this internal state and the input coordinates. The class also contains large, immutable static arrays (**Gradients2D**, **RandVecs3D**) used as pre-computed lookup tables for various algorithms.

- **Thread Safety:** **CRITICAL WARNING:** This class is **NOT thread-safe**. All configuration setters modify the internal state. Calling a setter from one thread while another thread is calling **getNoise** on the same instance will result in a race condition, producing non-deterministic and potentially corrupt output.

    All configuration **must** be completed before the instance is used for generation in a multithreaded environment. For parallel tasks like chunk generation, each worker thread should be provided with its own pre-configured, effectively immutable instance of **FastNoiseLite**. Sharing a single mutable instance across worker threads is an explicit anti-pattern and will lead to severe generation bugs.

## API Surface
The public API is divided into configuration methods (setters) and computation methods (**getNoise**, **DomainWarp**).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FastNoiseLite(int seed) | constructor | O(1) | Creates and initializes a new noise generator with a specific seed. |
| setSeed(int seed) | void | O(1) | Sets the master seed for all noise calculations. |
| setFrequency(float frequency) | void | O(1) | Sets the frequency, controlling the scale of the noise features. |
| setNoiseType(NoiseType type) | void | O(1) | Selects the core noise algorithm (e.g., OpenSimplex2, Perlin). |
| setFractalType(FractalType type) | void | O(1) | Selects the fractal algorithm for layering noise octaves. |
| setFractalOctaves(int octaves) | void | O(log N) | Sets the number of noise layers. Triggers a bounding recalculation. |
| getNoise(double x, double y) | float | O(octaves) | Samples the configured 2D noise field at the given coordinates. |
| getNoise(double x, double y, double z) | float | O(octaves) | Samples the configured 3D noise field at the given coordinates. |
| DomainWarp(Vector2 coord) | void | O(octaves) | Applies domain warp to the provided coordinate vector, modifying it in place. |

## Integration Patterns

### Standard Usage
The intended pattern is to instantiate, configure completely, and then reuse the object for all noise sampling related to that configuration.

```java
// 1. Create and configure the noise generator once.
FastNoiseLite terrainNoise = new FastNoiseLite(1337);
terrainNoise.setNoiseType(FastNoiseLite.NoiseType.OpenSimplex2);
terrainNoise.setFrequency(0.005f);
terrainNoise.setFractalType(FastNoiseLite.FractalType.FBm);
terrainNoise.setFractalOctaves(5);

// 2. Reuse the configured instance to generate a height map.
for (int x = 0; x < CHUNK_WIDTH; x++) {
    for (int z = 0; z < CHUNK_DEPTH; z++) {
        // The getNoise call is the hot path.
        float height = terrainNoise.getNoise(worldX + x, worldZ + z);
        // ... use height to place blocks
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call setter methods on a **FastNoiseLite** instance that is actively being used for generation by other threads. This is the most critical anti-pattern and will cause unpredictable world generation.

- **Per-Sample Instantiation:** Do not create a new **FastNoiseLite** object for every call to **getNoise**. The object is designed to be configured once and reused thousands or millions of times. Instantiating it inside a tight loop is extremely inefficient.
    ```java
    // BAD: Extremely inefficient object creation in a loop.
    for (int x = 0; x < 16; x++) {
        FastNoiseLite badNoise = new FastNoiseLite(1337); // DO NOT DO THIS
        float value = badNoise.getNoise(x, 0);
    }
    ```

## Data Pipeline
**FastNoiseLite** acts as a pure function in the context of a larger data flow, transforming coordinate inputs into noise values based on its internal configuration.

> Flow:
> World Generation System -> Provides (X, Y, Z) coordinates -> **FastNoiseLite** -> Applies internal state (Seed, Frequency, Algorithm) -> Performs mathematical computation -> Returns float value [-1, 1] -> World Generation System -> Maps float to game data (e.g., terrain height, biome ID)

