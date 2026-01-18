---
description: Architectural reference for IFloatCoordinateSupplier
---

# IFloatCoordinateSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Contract Interface

## Definition
```java
// Signature
public interface IFloatCoordinateSupplier {
```

## Architecture & Concepts
The IFloatCoordinateSupplier interface is a fundamental contract within the procedural generation library. It defines a standardized mechanism for retrieving a floating-point value based on a set of spatial coordinates and a seed. This abstraction is a cornerstone of the Strategy Pattern as applied to world generation, decoupling high-level generation algorithms (e.g., terrain sculpting, biome placement) from the underlying mathematical functions that produce the raw data (e.g., Perlin noise, Simplex noise, constant values).

This interface acts as the primary source for any procedurally generated, continuous data field. Implementations of this contract are responsible for the actual value calculation, allowing the engine to compose and chain different suppliers to create complex and varied environmental features. For example, one supplier might define the base terrain elevation, while another defines temperature, and a third defines humidity.

## Lifecycle & Ownership
As an interface, IFloatCoordinateSupplier itself has no lifecycle. The lifecycle and ownership semantics apply to the **concrete classes** that implement this contract.

- **Creation:** Instances of implementing classes are typically created and configured by a higher-level manager, such as a WorldGenerator or a BiomeDefinition. They are often instantiated based on world configuration files.
- **Scope:** The lifetime of a supplier instance is tied to its owner. A supplier used for chunk generation will live as long as the WorldGenerator for that dimension exists. A temporary supplier used for a specific structure might be garbage collected after the structure is generated.
- **Destruction:** There is no explicit destruction method. Instances are eligible for garbage collection when they are no longer referenced by any part of the generation system.

## Internal State & Concurrency
- **State:** This interface contract is inherently stateless. The `get` methods should, in principle, be pure functions where the output depends solely on the input parameters. However, concrete implementations may be stateful. For example, a caching noise implementation would contain a mutable cache.

- **Thread Safety:** **Not Guaranteed.** The interface itself provides no thread-safety guarantees. It is the responsibility of the concrete implementation to ensure safe concurrent access. Consumers of this interface **must not** assume that an instance is thread-safe unless explicitly documented by the implementing class. Calling `get` from multiple threads on a stateful, non-thread-safe implementation will lead to race conditions and non-deterministic world generation.

## API Surface
The public contract is minimal, focused exclusively on data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int seed, double x, double y) | float | *Varies* | Retrieves a 2D value. The complexity is entirely dependent on the implementation. |
| get(int seed, double x, double y, double z) | float | *Varies* | Retrieves a 3D value. The complexity is entirely dependent on the implementation. |

## Integration Patterns

### Standard Usage
The primary pattern is dependency injection, where a generator receives a pre-configured supplier instance and uses it to produce data.

```java
// A generator receives a supplier from a factory or configuration context
public class TerrainGenerator {
    private final IFloatCoordinateSupplier heightmapSupplier;

    public TerrainGenerator(IFloatCoordinateSupplier heightmapSupplier) {
        this.heightmapSupplier = heightmapSupplier;
    }

    public void generateChunk(int seed, ChunkCoordinate coord) {
        for (int x = 0; x < 16; x++) {
            for (int z = 0; z < 16; z++) {
                double worldX = coord.getWorldX(x);
                double worldZ = coord.getWorldZ(z);
                // Use the supplier to get the terrain height
                float height = heightmapSupplier.get(seed, worldX, worldZ);
                // ... use height to place blocks
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming Thread Safety:** Never share a single supplier instance across multiple generation threads unless its specific implementation is explicitly documented as thread-safe. This is a common source of subtle and difficult-to-reproduce generation bugs.
- **Type Checking:** Avoid using `instanceof` to check for a specific implementation of IFloatCoordinateSupplier. Doing so violates the principle of the abstraction and makes the system brittle. If special behavior is needed, it should be exposed through a different interface.
- **Ignoring the Seed:** The `seed` parameter is critical for reproducibility. Passing a constant or zero for the seed will result in identical output across different worlds, which is almost never the desired behavior.

## Data Pipeline
This interface serves as a data source at the beginning of many generation pipelines.

> Flow:
> Generator System -> Requests value for (Seed, X, Y, Z) -> **IFloatCoordinateSupplier Implementation** -> Returns float -> Voxel Type, Biome ID, or other derived data


