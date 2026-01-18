---
description: Architectural reference for ICoordinateDoubleSupplier
---

# ICoordinateDoubleSupplier

**Package:** com.hypixel.hytale.server.worldgen.util.function
**Type:** Functional Interface

## Definition
```java
// Signature
public interface ICoordinateDoubleSupplier {
```

## Architecture & Concepts
The ICoordinateDoubleSupplier interface is a foundational component of the server-side world generation pipeline. It defines a strict contract for any system that needs to provide a floating-point value based on a spatial coordinate. This is a classic implementation of the **Strategy Pattern**, designed to decouple world generation algorithms from the specific sources of data they consume.

In practice, this interface acts as a generic "value provider" for coordinates. Implementations can range from simple mathematical functions to complex, multi-layered noise generators like Perlin or Simplex noise. By abstracting the data source behind this interface, the core world generation logic can remain agnostic. For example, the same terrain sculpting algorithm can be fed values from a `PerlinNoiseSupplier` to create rolling hills or from a `CaveDensitySupplier` to carve out caverns, without changing the sculpting code itself.

The presence of two `apply` methods, one for 3D (x, y, z) and one for 4D (x, y, z, w) coordinates, indicates a highly flexible system. The 4D variant is typically used to introduce an additional dimension of variation, such as time, a seed offset, or another noise field, allowing for more complex and dynamic procedural generation.

## Lifecycle & Ownership
As an interface, ICoordinateDoubleSupplier does not have a lifecycle of its own. It is a stateless contract. The lifecycle and ownership concerns apply to the **concrete classes that implement** this interface.

- **Creation:** Implementations are typically instantiated and configured during the initialization of a specific `WorldGenerator` or a biome-specific generator. They are often created as anonymous classes or lambdas for simple functions, or as dedicated singleton services for complex, reusable noise fields.
- **Scope:** The lifetime of an implementing object is tied to the generator that owns it. For world-level noise functions, the instance may persist for the entire server session. For biome-specific functions, it may be created and destroyed as the relevant generation context becomes active or inactive.
- **Destruction:** Garbage collected when the owning generator is destroyed or goes out of scope. There are no explicit `close` or `destroy` methods on the contract, so implementations must not hold unmanaged resources.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, implementations may be stateful. For example, a noise supplier might cache results for recently queried coordinates to improve performance.

- **Thread Safety:** **CRITICAL.** World generation is a heavily multi-threaded process, with different threads often generating adjacent chunks concurrently. Any implementation of ICoordinateDoubleSupplier **must be unconditionally thread-safe**. If an implementation contains mutable state (e.g., a cache), all access to that state must be protected by appropriate synchronization mechanisms (e.g., locks, concurrent data structures). Failure to ensure thread safety will lead to severe data corruption, race conditions, and non-deterministic world generation. It is strongly recommended to design implementations to be fully stateless where possible.

## API Surface
The public contract consists of two methods for querying a value at a given coordinate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(int, int, int) | double | Varies | Supplies a value for a 3D coordinate (x, y, z). |
| apply(int, int, int, int) | double | Varies | Supplies a value for a 4D coordinate (x, y, z, w). |

**Warning:** The performance complexity of any `apply` implementation is paramount. These methods are called in tight loops, potentially millions of times per second during world generation. Any implementation with a complexity greater than O(1) or O(log N) will create a significant performance bottleneck. Avoid I/O, complex allocations, or heavy computations within this method.

## Integration Patterns

### Standard Usage
This interface is intended to be used as a dependency passed into higher-level world generation algorithms. Developers should implement the interface to encapsulate a specific noise function or data source and inject it into the generator.

```java
// A WorldGenerator receives an ICoordinateDoubleSupplier to define terrain height.
// The generator is unaware of the underlying noise algorithm (e.g., Perlin, Simplex).

public class TerrainGenerator {
    private final ICoordinateDoubleSupplier heightMap;

    public TerrainGenerator(ICoordinateDoubleSupplier heightMap) {
        this.heightMap = heightMap;
    }

    public void generateChunk(Chunk chunk) {
        for (int x = 0; x < 16; x++) {
            for (int z = 0; z < 16; z++) {
                // The generator simply applies the function to get the height.
                double height = heightMap.apply(chunk.getX() + x, 0, chunk.getZ() + z);
                // ... logic to place blocks based on height
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful, Unsynchronized Implementations:** Never create an implementation with mutable state that is not explicitly thread-safe. Sharing such an instance across chunk generation threads is a guaranteed source of bugs.
- **Blocking Operations:** Do not perform file I/O, network requests, or any other blocking operation within an `apply` method. This will stall the world generation thread and cause catastrophic performance degradation.
- **Complex Object Creation:** Avoid allocating new complex objects within `apply`. This method is on the hot path and excessive memory allocation will trigger frequent garbage collection, pausing the server.

## Data Pipeline
ICoordinateDoubleSupplier acts as a data **source** within the world generation pipeline. It does not transform data; it originates it on demand.

> Flow:
> World Generator Algorithm -> Requests value for coordinate (x,y,z) -> **ICoordinateDoubleSupplier.apply(x,y,z)** -> Returns double value (e.g., density, height, temperature) -> World Generator Algorithm uses value to determine block type or placement.

