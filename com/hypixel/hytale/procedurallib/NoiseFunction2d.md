---
description: Architectural reference for NoiseFunction2d
---

# NoiseFunction2d

**Package:** com.hypixel.hytale.procedurallib
**Type:** Functional Interface / Contract

## Definition
```java
// Signature
@FunctionalInterface
public interface NoiseFunction2d {
   double get(int var1, int var2, double var3, double var5);
}
```

## Architecture & Concepts
The NoiseFunction2d interface is a foundational contract within Hytale's procedural generation framework. It defines the standardized signature for any two-dimensional noise generation algorithm, such as Perlin, Simplex, or Worley noise.

This interface embodies the **Strategy Pattern**, allowing the world generation engine to be decoupled from the specific noise algorithm being used. High-level systems, like the TerrainGenerator or BiomePlacementService, operate against this interface, enabling developers to swap, chain, or modify noise implementations without altering the core generation logic. Its primary role is to transform a set of input coordinates and parameters into a deterministic, pseudo-random scalar value, which is then used to define features like terrain elevation, cave systems, temperature maps, or resource distribution.

**WARNING:** The parameter names in the source signature (var1, var2, etc.) are generic. Based on engine conventions, they typically map to:
- **var1:** A world or region seed.
- **var2:** A secondary seed or salt for a specific noise layer.
- **var3:** The X-coordinate.
- **var5:** The Z-coordinate.

Implementers must adhere to the expected parameter mapping of the system that will be consuming the function.

## Lifecycle & Ownership
As an interface, NoiseFunction2d does not have a lifecycle of its own. The lifecycle pertains to the concrete classes that *implement* this interface.

- **Creation:** Implementations (e.g., PerlinNoise, SimplexNoise) are typically instantiated and configured by a central procedural generation manager or a specific world generation task. They are often created in pools or as part of a larger "noise graph" during the initialization of a world generation session.
- **Scope:** The lifetime of a noise function instance is tied to the scope of the generation process it serves. For world generation, an instance may live for the duration of the entire world creation task. For smaller, transient procedural tasks, it may be short-lived.
- **Destruction:** Instances are subject to standard Java garbage collection. They are eligible for cleanup once the world generation task completes and all references to the function are released.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, concrete implementations are frequently stateful. For example, a Perlin noise implementation will hold a pre-computed, seeded permutation table to ensure deterministic output. This internal state is typically immutable after construction.
- **Thread Safety:** **CRITICAL:** All implementations of NoiseFunction2d **must be thread-safe**. The world generation engine heavily parallelizes chunk creation, and multiple worker threads will call the get method on a *single shared instance* of a noise function concurrently. Implementations must be re-entrant and contain no mutable state that is modified during the get call. All necessary context must be derived from the constructor-injected state (like the seed) or the method parameters.

## API Surface
The public contract consists of a single method, as mandated by the FunctionalInterface annotation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int, int, double, double) | double | O(1) | Computes and returns a noise value for a given 2D coordinate and seed. The output is typically normalized to a range like [-1.0, 1.0] or [0.0, 1.0]. |

## Integration Patterns

### Standard Usage
Implementations of this interface are injected into or retrieved by higher-level generation services. The service then invokes the get method repeatedly across a grid of coordinates to produce a coherent noise map.

```java
// A WorldGenerator retrieves a pre-configured noise function
// to generate a heightmap for a new chunk.
NoiseFunction2d heightmapNoise = noiseRegistry.get("terrain.base_height");
int chunkSize = 16;
double[] heightData = new double[chunkSize * chunkSize];

for (int x = 0; x < chunkSize; x++) {
    for (int z = 0; z < chunkSize; z++) {
        // Global world coordinates for deterministic, seamless noise
        double worldX = chunkOriginX + x;
        double worldZ = chunkOriginZ + z;
        
        // The world seed and a layer-specific salt are passed in
        heightData[z * chunkSize + x] = heightmapNoise.get(worldSeed, TERRAIN_SALT, worldX, worldZ);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Non-Deterministic Implementation:** An implementation that uses Math.random() or any other non-seeded random source is incorrect. It violates the core requirement of procedural generation, which is to produce the exact same world from the same seed.
- **Mutable Internal State:** Do not design an implementation that modifies its internal state (e.g., a field) within the get method. This will cause severe and difficult-to-debug race conditions when used by the multi-threaded world generator.
- **Ignoring Seed Parameters:** Failing to use the seed parameters (var1, var2) will result in every generated world being identical, defeating the purpose of a seed.

## Data Pipeline
NoiseFunction2d acts as a primary data source in the procedural generation pipeline. It does not typically process incoming data but rather originates it based on mathematical inputs.

> Flow:
> World Generator Request -> **NoiseFunction2d.get(seed, salt, x, z)** -> Scalar double value -> Aggregation into Heightmap/Biomemap -> Voxel Block Placement

