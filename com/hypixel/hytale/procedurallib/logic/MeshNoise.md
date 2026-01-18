---
description: Architectural reference for MeshNoise
---

# MeshNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Transient

## Definition
```java
// Signature
public class MeshNoise implements NoiseFunction {
```

## Architecture & Concepts
MeshNoise is a specialized implementation of the NoiseFunction interface that generates a procedural web or mesh-like pattern. It operates by calculating the shortest distance from a given point to the nearest line segment connecting adjacent, jittered grid points. This technique is a variation of Worley or Cellular noise, but focuses on the edges of the Voronoi cells rather than the distance to the cell's center point.

This class serves as a fundamental building block within the procedural generation library. It is designed to be composed with other functions to create complex material distributions, biome boundaries, or cave-like structures. Its output, a scalar field ranging from -1.0 to 1.0, represents the proximity to the mesh lines, where values near -1.0 are on a line and values near 1.0 are furthest away.

The behavior of the mesh is controlled by three primary parameters:
1.  **Density:** An IIntCondition that determines whether a grid cell contains an active point. This allows for creating sparse or broken mesh patterns.
2.  **Thickness:** Controls the width of the generated lines in the mesh.
3.  **Jitter:** Introduces randomness to the grid points, transforming a regular grid into a more organic, irregular pattern.

## Lifecycle & Ownership
-   **Creation:** Instantiated directly via its public constructor, typically during the configuration phase of a larger procedural generation system (e.g., a WorldGenerator or BiomeProvider). All dependencies and configuration parameters are injected at creation time.
-   **Scope:** The lifetime of a MeshNoise instance is bound to its owner. It is designed to be created once with a specific configuration and reused for millions of calculations. It holds no state between calls and can persist for as long as its configuration is needed.
-   **Destruction:** Managed by the Java Garbage Collector. As a plain Java object with no native resources, it is automatically reclaimed when no longer referenced.

## Internal State & Concurrency
-   **State:** **Immutable.** All member fields are declared final and are initialized exclusively in the constructor. The object's configuration cannot be changed after instantiation. Each call to the get method is a pure function with no side effects.
-   **Thread Safety:** **Fully thread-safe.** Due to its immutable nature, a single MeshNoise instance can be safely shared and accessed by multiple threads concurrently without any external synchronization. This is a critical design feature for enabling high-performance, parallelized world generation.

## API Surface
The public contract is defined by the NoiseFunction interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, offsetSeed, x, y) | double | O(1) | Calculates the 2D mesh noise value for a given coordinate. The algorithm's complexity is constant as it only ever inspects a fixed 3x3 grid of cells. |
| get(seed, offsetSeed, x, y, z) | double | O(1) | **Unsupported.** This function is not implemented for 3D noise. Throws UnsupportedOperationException upon invocation. |

## Integration Patterns

### Standard Usage
A MeshNoise instance should be configured and created once, then reused for all subsequent noise sampling. It is a computational object, not a service.

```java
// 1. Define a density condition (e.g., 50% of cells are active)
IIntCondition density = new IntThresholdCondition(0.5);

// 2. Instantiate MeshNoise with the desired configuration
double thickness = 0.1;
double jitter = 0.8;
NoiseFunction mesh = new MeshNoise(density, thickness, jitter, jitter);

// 3. Reuse the instance to sample noise values
int seed = 12345;
double value = mesh.get(seed, 0, 150.75, -45.2);
```

### Anti-Patterns (Do NOT do this)
-   **Per-Call Instantiation:** Never create a new MeshNoise instance inside a loop or for every point you need to sample. This is highly inefficient and will cause significant performance degradation due to excessive object allocation.

    ```java
    // BAD: Creates an object for every single point
    for (int x = 0; x < 100; x++) {
        NoiseFunction badMesh = new MeshNoise(density, 0.1, 0.8, 0.8); // ANTI-PATTERN
        double value = badMesh.get(seed, 0, x, 0);
    }
    ```

-   **3D Invocation:** Do not attempt to use this implementation for 3D noise generation. The `get(x, y, z)` method is explicitly unsupported and will crash the calling thread.

## Data Pipeline
MeshNoise functions as a stateless transformation stage within a larger data processing pipeline, typically for world generation.

> Flow:
> Generator Request (Coordinates) -> **MeshNoise.get(x, y)** -> Scalar Noise Value [-1, 1] -> Voxel/Material Selection Logic -> Final World Data

