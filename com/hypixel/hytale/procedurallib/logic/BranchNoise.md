---
description: Architectural reference for BranchNoise
---

# BranchNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Transient

## Definition
```java
// Signature
public class BranchNoise implements NoiseFunction {
```

## Architecture & Concepts
BranchNoise is a specialized, composite implementation of the NoiseFunction interface. It is designed to generate complex, organic, vein-like, or branching patterns. It is not a general-purpose noise generator like Perlin or Simplex noise, but rather a higher-level function used to orchestrate other noise primitives for a specific structural outcome, such as rivers, cave networks, or lightning effects.

The core architectural concept is a two-pass evaluation strategy:

1.  **Parent Feature Pass:** The algorithm first uses a `CellDistanceFunction` (parentFunction) to establish a sparse grid of "parent" feature points. For any given input coordinate, it identifies the nearest parent point. A configurable density condition (`parentDensity`) acts as a filter, culling potential branches and creating a more irregular, natural distribution.

2.  **Line Generation Pass:** If a parent point exists for a region, a second `CellDistanceFunction` (lineFunction) calculates the distance to a conceptual line or tube originating from that parent point. The thickness of this line is not constant; it is dynamically modulated by the distance from the parent feature's core, controlled by the `lineThickness` range.

The final output value is a smooth interpolation between a base parent value and the calculated line value. This blending, controlled by the `parentFade` range, creates soft falloffs from the main branch, preventing hard edges.

## Lifecycle & Ownership
-   **Creation:** BranchNoise instances are not intended for direct instantiation in gameplay code. They are constructed by a higher-level procedural generation service or configuration loader during the world generation setup phase. The numerous constructor parameters are typically deserialized from a data file (e.g., JSON) that defines the complete noise profile for a biome or feature.

-   **Scope:** An instance of BranchNoise is an immutable configuration object. Its lifetime is tied to the world generation asset set it belongs to. It persists as long as that configuration is loaded and can be reused indefinitely for generation tasks.

-   **Destruction:** The object is managed by the Java garbage collector. No explicit destruction or resource cleanup is necessary. It is reclaimed once the world generation configuration that created it is unloaded.

## Internal State & Concurrency
-   **State:** Immutable. All internal fields are `final` and are dependency-injected via the constructor. An instance of BranchNoise represents a fixed, unchangeable noise algorithm. It does not cache results or modify its internal state after creation.

-   **Thread Safety:** **CRITICAL WARNING:** This class is **NOT THREAD-SAFE**. While the object's configuration state is immutable, the core `get` method relies on a shared, static, and mutable `ResultBuffer.buffer2d` for performance. Concurrent calls to the `get` method from multiple threads will cause race conditions and lead to severe data corruption and unpredictable noise output.

    This design assumes that the procedural generation pipeline processes chunks or tasks in a single-threaded manner or provides a thread-local context for such buffers. Any system invoking this class must enforce a strict single-threaded execution context for calls to the `get` method.

## API Surface
The public API is minimal, adhering to the `NoiseFunction` interface contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, offsetSeed, x, y) | double | O(K) | Computes the 2D noise value. K is the number of neighboring cells evaluated by the underlying CellDistanceFunctions. Throws no exceptions. |
| get(seed, offsetSeed, x, y, z) | double | N/A | **Unsupported.** Throws UnsupportedOperationException. This function is strictly 2D. |

## Integration Patterns

### Standard Usage
BranchNoise should be treated as an implementation detail of the `NoiseFunction` interface. It is retrieved from a central registry or configuration manager and used to generate values for a specific region.

```java
// A hypothetical WorldGenerator retrieving a pre-configured NoiseFunction
NoiseFunction riverNoise = worldConfig.getNoiseFunction("biome.river.branch_pattern");

for (int x = 0; x < CHUNK_SIZE; x++) {
    for (int y = 0; y < CHUNK_SIZE; y++) {
        double worldX = chunkOriginX + x;
        double worldY = chunkOriginY + y;

        // The get method is called within a single-threaded generation task
        double noiseValue = riverNoise.get(worldSeed, 0, worldX, worldY);

        if (noiseValue > 0.8) {
            // Place water block
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually construct BranchNoise using `new`. The complexity of its constructor is a strong indicator that it should be data-driven and managed by a framework. Manual creation is brittle and circumvents the intended configuration-driven design.

-   **Concurrent Execution:** Never call the `get` method from multiple threads simultaneously. This will corrupt the shared internal buffer and produce incorrect results. All calls must be serialized or executed in a context that guarantees thread-isolation for the `ResultBuffer`.

    ```java
    // BAD: Causes race conditions
    Parallel.forEach(coordinates, (coord) -> {
        double value = branchNoise.get(seed, 0, coord.x, coord.y); // DANGER
        // ...
    });
    ```

-   **3D Usage:** Do not attempt to use this function for 3D noise generation. The 3D `get` method will always throw an `UnsupportedOperationException`.

## Data Pipeline
The flow of data through the `get` method is a multi-stage process that transforms input coordinates into a final noise value.

> Flow:
> (x, y) Coordinates -> Parent `CellDistanceFunction` -> Parent Point & Hash -> **BranchNoise** (Density Check & Filtering) -> Line `CellDistanceFunction` -> Raw Line Distance -> **BranchNoise** (Blending, Fading, & Normalization) -> Final Noise Value [-1.0, 1.0]

