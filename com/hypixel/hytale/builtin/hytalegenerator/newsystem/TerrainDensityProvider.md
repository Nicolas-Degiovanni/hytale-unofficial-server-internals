---
description: Architectural reference for TerrainDensityProvider
---

# TerrainDensityProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem
**Type:** Functional Interface / Strategy

## Definition
```java
// Signature
@FunctionalInterface
public interface TerrainDensityProvider {
   double get(@Nonnull Vector3i var1, @Nonnull WorkerIndexer.Id var2);
}
```

## Architecture & Concepts
The TerrainDensityProvider interface is a fundamental contract within the procedural world generation system. It embodies the **Strategy Pattern**, defining a single, critical behavior: determining the density of the world at any given 3D integer coordinate. It does not generate blocks directly; instead, it provides a scalar field, often called an isosurface, which other systems use to construct the actual geometry.

Architecturally, this interface decouples the high-level mesh generation algorithm (like Marching Cubes) from the specific mathematical formulas that define the shape of the world. This allows the engine to plug in different implementations to create vastly different world shapes—from smooth, rolling hills to complex cave systems or floating islands—without altering the core meshing pipeline.

The density value itself is a convention:
-   **Negative values** typically represent air or empty space.
-   **Positive values** represent solid material.
-   **Zero** represents the exact boundary or surface of the terrain.

The inclusion of a `WorkerIndexer.Id` in the `get` method signature is a critical design choice for performance. It signals that the density calculation is expected to be performed in a highly parallel environment. Implementations can use this ID to access thread-local caches, noise generators, or other resources, avoiding lock contention and ensuring deterministic output across multiple worker threads.

### Lifecycle & Ownership
As a functional interface, TerrainDensityProvider does not have a lifecycle of its own. Its *implementations* are subject to the following lifecycle:

-   **Creation:** Implementations, whether as dedicated classes or lambdas, are instantiated by higher-level generator systems, such as a `BiomeManager` or a `WorldGenerator`, during the setup phase of a world generation task.
-   **Scope:** An instance of a provider typically exists for the duration of a specific generation task, such as generating a single chunk or a region. It is passed by reference to the worker threads that need it.
-   **Destruction:** Implementations are managed by the Java Garbage Collector. They are eligible for cleanup once the generation task that created them is complete and no further references exist.

## Internal State & Concurrency
-   **State:** The interface is stateless. Implementations are strongly encouraged to be immutable or to manage state in a thread-safe manner. Any mutable state, such as a cache, should be keyed by the `WorkerIndexer.Id` to ensure thread isolation.
-   **Thread Safety:** Implementations of this interface **must be thread-safe**. The world generation engine will call the `get` method concurrently from multiple worker threads. Failure to ensure thread safety will result in data races, non-deterministic world generation, and potential crashes. The use of thread-local resources is the preferred pattern for managing state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(Vector3i position, WorkerIndexer.Id workerId) | double | O(N) | Calculates the terrain density at a given world coordinate. Complexity is dependent on the implementation, typically proportional to the number of noise octaves (N) sampled. |

## Integration Patterns

### Standard Usage
The provider is typically passed as a dependency to a meshing algorithm, which queries it repeatedly to build geometry for a volume of space.

```java
// A simplified mesher using the provider
public class ChunkMesher {
    public Mesh buildMesh(TerrainDensityProvider densityProvider, WorkerIndexer.Id workerId) {
        // ... setup ...
        for (int x = 0; x < 16; x++) {
            for (int y = 0; y < 16; y++) {
                for (int z = 0; z < 16; z++) {
                    Vector3i currentPos = new Vector3i(x, y, z);
                    // Query the density field
                    double density = densityProvider.get(currentPos, workerId);
                    // Use density to generate mesh vertices...
                }
            }
        }
        // ... return mesh ...
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring the Worker ID:** An implementation that uses a shared, non-thread-safe resource (e.g., a single `java.util.Random` instance) and ignores the `workerId` is fundamentally broken. This will cause severe race conditions and produce inconsistent, artifact-ridden terrain.
-   **Shared Mutable State:** Avoid implementations that rely on `synchronized` blocks for performance-critical paths. The overhead of locking in the tight loop of a generator is prohibitive. Use thread-local storage or other lock-free techniques.

## Data Pipeline
This component acts as a source of procedural data at the beginning of the geometry generation pipeline.

> Flow:
> **TerrainDensityProvider** -> Marching Cubes Algorithm -> Raw Mesh Data -> Mesh Optimizer -> Voxel Chunk Geometry

