---
description: Architectural reference for IDoubleCoordinateHashSupplier
---

# IDoubleCoordinateHashSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Functional Interface / Contract

## Definition
```java
// Signature
public interface IDoubleCoordinateHashSupplier {
   double get(int var1, int var2, int var3, long var4);
}
```

## Architecture & Concepts
The IDoubleCoordinateHashSupplier interface defines a fundamental contract within the procedural generation engine. It serves as a standardized source for deterministic, pseudo-random values based on a spatial coordinate and a seed. This abstraction is critical for decoupling world generation algorithms from the specific noise or hashing functions they use.

This interface represents a pure function: for a given set of inputs (a 3D integer coordinate and a 64-bit seed), it must always produce the exact same double-precision floating-point output. This determinism is the cornerstone of Hytale's ability to generate consistent and reproducible worlds.

Implementations of this interface may range from simple mathematical hash functions to complex, layered noise algorithms like Perlin or Simplex noise. The consumer of the interface, such as a BiomeGenerator or TerrainCarver, operates on the supplied value without needing knowledge of the underlying implementation.

## Lifecycle & Ownership
As an interface, IDoubleCoordinateHashSupplier does not have a lifecycle of its own. The lifecycle pertains to the concrete classes that implement it.

- **Creation:** Implementations are typically instantiated by a WorldGenerationContext or a specialized NoiseFactory at the beginning of a world generation process. They are often configured and composed into chains or layers to produce the final desired noise characteristics.
- **Scope:** The lifetime of an implementing object is tied to a specific generation task. For example, a single instance may be created and passed to all generators responsible for a single world chunk column to ensure consistency. It is generally short-lived and task-scoped.
- **Destruction:** The object is eligible for garbage collection once the generation task (e.g., chunk generation) that holds a reference to it is complete. There is no manual destruction or cleanup required by the contract.

## Internal State & Concurrency
- **State:** The contract is implicitly stateless. The `get` method's signature contains all necessary information (coordinate and seed) to compute the value. Implementations **should not** rely on internal mutable state to calculate their output. While an implementation *could* cache results for performance, this must be done in a thread-safe manner and must not alter the deterministic nature of the output.
- **Thread Safety:** The interface itself makes no guarantees. However, the functional design strongly implies that implementations **must be thread-safe**. World generation is a highly parallelized process, and a single supplier instance will be called concurrently from multiple worker threads processing different parts of the world. Stateless implementations are inherently thread-safe.

**WARNING:** Stateful implementations of this interface are a significant source of concurrency bugs and non-deterministic world generation. Avoid them unless absolutely necessary, and if so, proper synchronization (e.g., using ConcurrentHashMap for a cache) is mandatory.

## API Surface
The parameter names `var1`, `var2`, `var3`, and `var4` in the source correspond to `x`, `y`, `z`, and `seed` respectively.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int x, int y, int z, long seed) | double | Implementation-dependent | Returns a deterministic, pseudo-random double value for a given 3D coordinate and world seed. The output range (e.g., [0.0, 1.0) vs [-1.0, 1.0]) is defined by the specific implementation. |

## Integration Patterns

### Standard Usage
This interface is consumed by higher-level procedural systems. It is typically retrieved from a context or factory and used to drive generation logic.

```java
// A generator receives a supplier, not a concrete implementation
public void generateTerrain(int chunkX, int chunkZ, IDoubleCoordinateHashSupplier heightMapSupplier) {
    long worldSeed = this.context.getWorldSeed();

    for (int x = 0; x < 16; x++) {
        for (int z = 0; z < 16; z++) {
            int worldX = chunkX * 16 + x;
            int worldZ = chunkZ * 16 + z;

            // Get a deterministic height value for this column
            double heightValue = heightMapSupplier.get(worldX, 0, worldZ, worldSeed);

            // Use the value to determine the terrain height
            int terrainHeight = 64 + (int)(heightValue * 32);
            setTopBlock(worldX, terrainHeight, worldZ, Blocks.GRASS);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming Output Range:** Do not assume the output of `get` is normalized to a specific range like [0.0, 1.0). One implementation might return values in [-1.0, 1.0), while another might have a much larger range. The consuming code is responsible for mapping or normalizing the value as needed.
- **Ignoring the Seed:** Failing to pass the correct world seed will result in worlds that are not reproducible. The seed is a mandatory parameter for ensuring determinism.
- **Stateful Implementations:** Creating an implementation that relies on an internal counter or other mutable state will break determinism and cause severe issues in a multithreaded environment.

## Data Pipeline
The flow of data is straightforward, typically initiated by a world generator and terminating in a decision about the game world's structure.

> Flow:
> World Generator needs data for coordinate (x, y, z) -> Calls **IDoubleCoordinateHashSupplier.get(x, y, z, seed)** -> Receives `double` value -> Generator logic maps the value to a Block Type, Biome ID, or other world feature.

