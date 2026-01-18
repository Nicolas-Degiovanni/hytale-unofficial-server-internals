---
description: Architectural reference for NoiseFunction3d
---

# NoiseFunction3d

**Package:** com.hypixel.hytale.procedurallib
**Type:** Functional Interface / Strategy

## Definition
```java
// Signature
@FunctionalInterface
public interface NoiseFunction3d {
   double get(int var1, int var2, double var3, double var5, double var7);
}
```

## Architecture & Concepts

The NoiseFunction3d interface is a foundational component of Hytale's procedural generation engine. It formalizes the contract for any algorithm that can produce a scalar noise value for a given point in 3D space. Architecturally, it serves as a classic **Strategy Pattern**, decoupling the high-level world generation orchestrators (e.g., TerrainGenerator, BiomeSampler) from the specific mathematical implementations of noise (e.g., Perlin, Simplex, Worley).

This abstraction is critical for creating a flexible and extensible world generation system. It allows designers and engineers to compose complex terrain features by layering, combining, and modifying different noise function implementations without altering the core generation logic.

The `get` method's signature is highly optimized for performance in large-scale world generation. The integer parameters (`var1`, `var2`) typically represent large-scale grid coordinates, such as chunk X and Z, while the double parameters represent finer-grained local coordinates and other noise-shaping values. This mixed-precision approach mitigates floating-point inaccuracies over vast distances from the world origin.

## Lifecycle & Ownership

As a functional interface, NoiseFunction3d itself does not have a lifecycle. The following pertains to its concrete **implementations**.

-   **Creation:** Implementations are typically instantiated and configured during the server's world-generation initialization phase. They are often constructed based on world configuration files (e.g., defining a biome's specific noise characteristics like seed, frequency, and octaves).
-   **Scope:** A given noise function instance is typically stateless and immutable after configuration. This allows a single instance to be shared across multiple threads and used for the entire duration of a world generation process. They are effectively world-scoped, long-lived objects.
-   **Destruction:** Instances are eligible for garbage collection when the world generation context is torn down or a world is unloaded from memory.

## Internal State & Concurrency

-   **State:** The interface contract implies statelessness. Any conforming implementation of the `get` method should be a **pure function**—its output must depend solely on its input parameters. Implementations may hold configuration data (e.g., seed, permutation tables) but this data must be immutable after initialization.

-   **Thread Safety:** **CRITICAL:** All implementations of NoiseFunction3d **must be unconditionally thread-safe**. The world generation system heavily parallelizes chunk creation, and the `get` method will be invoked concurrently from many worker threads. Implementations must be re-entrant and must not contain any mutable shared state, locks, or other synchronization primitives that would create contention. Failure to adhere to this will result in severe performance degradation and non-deterministic, corrupt world generation.

## API Surface

The public contract consists of a single method, defining the functional signature.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int chunkX, int chunkZ, double y, double localX, double localZ) | double | O(k) | Computes the noise value at a specific world coordinate. The complexity is dependent on the implementation, where k is often the number of octaves. |

**Parameter Interpretation (Inferred):**
-   `var1`: The integer X-coordinate of the chunk or region.
-   `var2`: The integer Z-coordinate of the chunk or region.
-   `var3`: The absolute Y-coordinate (height).
-   `var5`: The local, fractional X-offset within the chunk/region.
-   `var7`: The local, fractional Z-offset within the chunk/region.

## Integration Patterns

### Standard Usage

Developers should not invoke a NoiseFunction3d directly. Instead, they should implement the interface to define a new noise behavior and register it with the world generation framework. The most common implementation pattern is a lambda or a dedicated class.

```java
// Example of a simple, compliant implementation for use in a worldgen profile.
// This function would be passed to a terrain generator, not called directly.

NoiseFunction3d simpleCaveNoise = (chunkX, chunkZ, y, localX, localZ) -> {
    // Combine inputs to form a single point for a 3D noise library
    double worldX = (chunkX * 16.0) + localX;
    double worldZ = (chunkZ * 16.0) + localZ;

    // NOTE: Assumes the existence of a thread-safe noise utility
    return SimplexNoise.get(worldX * 0.05, y * 0.05, worldZ * 0.05);
};

// The framework would then use this function:
// terrainGenerator.setCaveNoise(simpleCaveNoise);
```

### Anti-Patterns (Do NOT do this)

-   **Stateful Implementations:** Do not create implementations that modify internal fields within the `get` method. This will break thread safety and lead to unpredictable world generation.

    ```java
    // DO NOT DO THIS
    class BadNoise implements NoiseFunction3d {
        private double lastValue = 0.0; // Mutable state
        public double get(...) {
            this.lastValue += 0.1; // Side effect, not thread-safe
            return Perlin.get(...) + this.lastValue;
        }
    }
    ```

-   **Ignoring Coordinate System:** Do not assume the parameters map directly to `x, y, z`. Misinterpreting the chunk vs. local coordinate separation will result in seams between chunks or distorted noise patterns.

## Data Pipeline

The NoiseFunction3d is a core processing step in the transformation of world coordinates into final block data.

> Flow:
> World Coordinate Request -> **NoiseFunction3d Implementation** -> Raw Noise Value [-1.0, 1.0] -> Domain Warping / Post-Processing -> Terrain Density Value -> Block Type Selection -> Chunk Data Buffer

