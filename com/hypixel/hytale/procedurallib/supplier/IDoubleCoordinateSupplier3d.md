---
description: Architectural reference for IDoubleCoordinateSupplier3d
---

# IDoubleCoordinateSupplier3d

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Functional Interface / Contract

## Definition
```java
// Signature
public interface IDoubleCoordinateSupplier3d {
```

## Architecture & Concepts
The IDoubleCoordinateSupplier3d interface is a foundational contract within Hytale's procedural generation library. It defines a standardized mechanism for sampling a value from a four-dimensional input space, comprising a 3D spatial coordinate (x, y, z) and an integer seed.

This abstraction is critical for decoupling world generation algorithms from the specific noise functions or data sources they rely on. By programming against this interface, a terrain generator, for instance, can be configured to use Perlin noise, Simplex noise, a constant value, or a complex composition of multiple suppliers without changing its core logic. It represents a single, queryable point in a procedural data field.

This interface is the 3D equivalent of other suppliers in the library and is primarily used for volumetric data generation, such as 3D density fields for caves, ore distribution, or temperature gradients.

## Lifecycle & Ownership
As an interface, IDoubleCoordinateSupplier3d does not have a lifecycle of its own. The lifecycle and ownership semantics apply to the concrete classes that **implement** this interface.

- **Creation:** Implementations are typically instantiated and configured during the initialization of a world generation pipeline. For example, a BiomeGenerator might create and hold a reference to a PerlinNoiseSupplier that implements this contract.
- **Scope:** The lifetime of an implementation is tied to its owner. It generally persists for the duration of a specific world generation session or for as long as the parent generator object is alive.
- **Destruction:** Implementations are subject to standard Java garbage collection once their owning object is destroyed and all references are released. They do not typically manage unmanaged resources and require no explicit cleanup.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Implementations, however, may be stateful (e.g., caching pre-computed noise values) or stateless (e.g., a pure mathematical function). The contract makes no assumptions about the state of the implementation.
- **Thread Safety:** **CRITICAL:** Implementations of this interface **must be thread-safe**. World generation is a heavily multi-threaded process. The get method will be called concurrently from multiple worker threads processing different regions of the world. Any mutable state within an implementation must be protected by appropriate synchronization mechanisms (e.g., locks, concurrent data structures) to prevent race conditions and ensure deterministic output. Failure to do so will result in corrupted or non-reproducible worlds.

## API Surface
The public contract consists of a single method for sampling the data field.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int seed, double x, double y, double z) | double | O(1) | Samples the field at the given 4D coordinate. The complexity is O(1) from the caller's perspective, but the actual cost depends on the implementation (e.g., multi-octave noise functions). |

## Integration Patterns

### Standard Usage
The interface is intended to be used as a dependency for higher-level generation algorithms. The algorithm requests a supplier and invokes its get method within its generation loop.

```java
// A generator receives a supplier, often via its constructor or a factory.
IDoubleCoordinateSupplier3d densitySource = generatorContext.getDensitySupplier();

// The generator iterates over a volume, sampling the supplier for each point.
for (int x = 0; x < 16; x++) {
    for (int y = 0; y < 256; y++) {
        for (int z = 0; z < 16; z++) {
            double worldX = chunk.x * 16 + x;
            double worldY = y;
            double worldZ = chunk.z * 16 + z;

            // Sample the density value using the abstract supplier
            double density = densitySource.get(worldSeed, worldX, worldY, worldZ);

            // Use the density value to determine block type
            if (density > 0.5) {
                setBlock(x, y, z, Block.STONE);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Seed:** An implementation that disregards the seed parameter is fundamentally broken. It will produce the same output for every world, defeating the purpose of procedural generation. All internal calculations must be derived from the provided seed.
- **Unsynchronized State:** Implementing this interface with a class that contains mutable fields (e.g., a cache) without making access to those fields thread-safe is a severe error. This will lead to non-deterministic world generation and hard-to-debug race conditions.

## Data Pipeline
This interface acts as a data source, not a processing stage. It is the origination point for procedural values within a larger generation pipeline.

> Flow:
> World Generator Algorithm -> Requests value for (seed, x, y, z) -> **IDoubleCoordinateSupplier3d Implementation** -> Computes or retrieves value -> Returns double -> Generator uses value to place blocks/data

