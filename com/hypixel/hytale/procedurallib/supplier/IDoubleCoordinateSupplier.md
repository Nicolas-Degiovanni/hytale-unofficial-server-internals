---
description: Architectural reference for IDoubleCoordinateSupplier
---

# IDoubleCoordinateSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IDoubleCoordinateSupplier {
```

## Architecture & Concepts
The IDoubleCoordinateSupplier interface defines a fundamental contract within the procedural generation library. It standardizes the mechanism for retrieving a double-precision floating-point value based on a set of spatial coordinates and a seed.

This abstraction is critical for decoupling procedural generation algorithms (e.g., terrain heightmap generation, biome placement, 3D noise fields) from the specific noise functions or data sources they rely on. Consumers of this interface request a value for a given coordinate, and the concrete implementation provides it. This allows for interchangeable noise algorithms—such as Perlin, Simplex, or Worley—without altering the core generation logic. It serves as a foundational component in the procedural data pipeline, acting as the primary source for continuous, coordinate-based data.

## Lifecycle & Ownership
As an interface, IDoubleCoordinateSupplier does not have its own lifecycle. The lifecycle is entirely determined by the concrete class that implements this contract and the system that manages that implementation.

- **Creation:** Concrete implementations (e.g., a PerlinNoiseSupplier) are typically instantiated by a higher-level manager, such as a WorldGenerator or a ProceduralContext, during the setup phase of a generation task.
- **Scope:** The lifetime of an implementing object is usually tied to the scope of the generation process it serves. It may be short-lived for a single chunk generation or persist for the duration of a full world creation session.
- **Destruction:** The object is eligible for garbage collection when the owning generator or context is destroyed.

**WARNING:** Developers must not make assumptions about the lifecycle of a given supplier. Always retrieve instances from the appropriate context or factory.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, implementations may be stateful. For example, a noise supplier might cache its seed and permutation tables, or a more advanced supplier could cache previously computed results for performance.
- **Thread Safety:** The contract does not enforce thread safety. It is the **absolute responsibility of the implementing class** to guarantee safe concurrent access. Given that procedural generation is a highly parallelized task, implementations of this interface are expected to be thread-safe or immutable. Calls to the get method should not produce side effects that interfere with other threads.

**WARNING:** Calling get on a non-thread-safe implementation from multiple threads will lead to data corruption, race conditions, and non-deterministic generation results.

## API Surface
The public contract consists of two methods for querying values in 2D and 3D space, respectively.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int seed, double x, double y) | double | Implementation-Defined | Retrieves a 2D value for the given seed and coordinates. |
| get(int seed, double x, double y, double z) | double | Implementation-Defined | Retrieves a 3D value for the given seed and coordinates. |

## Integration Patterns

### Standard Usage
The interface is intended to be used as a dependency provided to a generation algorithm. The algorithm iterates over a coordinate space and uses the supplier to sample values at each point.

```java
// A generator receives a supplier, abstracting the noise source.
public class TerrainGenerator {
    private final IDoubleCoordinateSupplier heightSupplier;

    public TerrainGenerator(IDoubleCoordinateSupplier heightSupplier) {
        this.heightSupplier = heightSupplier;
    }

    public void generateChunk(int seed, int chunkX, int chunkZ) {
        for (int x = 0; x < 16; x++) {
            for (int z = 0; z < 16; z++) {
                double worldX = chunkX * 16 + x;
                double worldZ = chunkZ * 16 + z;
                // The generator is agnostic to the underlying noise function.
                double height = heightSupplier.get(seed, worldX, worldZ);
                // ... use height to place blocks
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Implementation-Checking:** Do not use `instanceof` to check for a specific implementation of IDoubleCoordinateSupplier. This breaks the abstraction and creates brittle, tightly-coupled code.
- **Assuming Statelessness:** Do not assume an implementation is stateless. While the contract is stateless, an implementation may perform internal caching or other stateful operations.
- **Ignoring the Seed:** Do not pass a constant or zero value for the seed unless you explicitly intend for every generation to be identical. The seed is critical for reproducibility and variation.

## Data Pipeline
This interface acts as a data source node in a larger procedural generation graph.

> Flow:
> Generation Algorithm -> Requests value for coordinate (x, y) -> **IDoubleCoordinateSupplier.get()** -> Concrete Noise Function (e.g., Perlin) -> Returns double value -> Generation Algorithm uses value for placement logic

