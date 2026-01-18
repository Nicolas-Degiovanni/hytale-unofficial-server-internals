---
description: Architectural reference for IFloatCoordinateHashSupplier
---

# IFloatCoordinateHashSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Functional Interface / Contract

## Definition
```java
// Signature
public interface IFloatCoordinateHashSupplier {
   float get(int var1, double var2, double var4, long var6);
}
```

## Architecture & Concepts
The IFloatCoordinateHashSupplier interface defines a strict contract for any system that provides a deterministic, pseudo-random floating-point value based on spatial coordinates and a seed. It is a foundational component of Hytale's procedural generation engine.

This interface acts as an abstraction layer, decoupling high-level world generation algorithms (e.g., terrain heightmap generation, biome placement) from the underlying noise or hashing function. This allows the engine to swap different noise implementations (such as Perlin, Simplex, or custom hashing functions) without altering the generator logic that consumes them.

Its primary role is to ensure that for a given set of inputs—coordinates and a world seed—the output is always identical. This determinism is critical for generating consistent and reproducible worlds.

## Lifecycle & Ownership
As an interface, IFloatCoordinateHashSupplier does not have a lifecycle of its own. The lifecycle is entirely dictated by its concrete implementations.

- **Creation:** Implementations are typically instantiated and configured during the initialization of a world's `ProceduralGenerationContext`. They are often created as stateless, reusable objects.
- **Scope:** The scope of an implementation is tied to the world generation session it is associated with. It may be scoped to a specific generator (e.g., a cave generator) or be globally available for the duration of the world's creation.
- **Destruction:** Implementations are subject to standard Java garbage collection once the `ProceduralGenerationContext` and any systems holding a reference to them are destroyed.

## Internal State & Concurrency
- **State:** The contract is implicitly stateless. Implementations are expected to behave as pure functions, where the output of the `get` method depends solely on its input arguments. While an implementation *may* use internal caches for performance, this is an implementation detail and must not alter the deterministic nature of the output.
- **Thread Safety:** Implementations of this interface **must be thread-safe**. The procedural generation engine heavily relies on multi-threading to generate world chunks in parallel. Any implementation must guarantee that concurrent calls to the `get` method from multiple threads will not result in race conditions or inconsistent state. This is typically achieved by avoiding mutable shared state.

## API Surface
The API surface consists of a single method, defining the core function of the contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int type, double x, double z, long seed) | float | Varies | Returns a deterministic float hash for a given 2D coordinate. The complexity depends entirely on the underlying noise algorithm of the implementation. |

**WARNING:** The output range of the returned float is not defined by this contract. Consumers must not assume a specific range (e.g., 0.0 to 1.0). If a normalized value is required, it is the responsibility of the calling system to perform the normalization.

## Integration Patterns

### Standard Usage
This interface is not used directly but is provided to other procedural systems that require a source of deterministic noise. A generator retrieves a specific implementation from a context or registry and calls its `get` method repeatedly for different coordinates.

```java
// A terrain generator receives a supplier during its initialization
// and uses it to determine block heights.
IFloatCoordinateHashSupplier heightSupplier = proceduralContext.getSupplier("terrain_height");
long worldSeed = world.getSeed();

for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        double worldX = chunk.getOriginX() + x;
        double worldZ = chunk.getOriginZ() + z;

        // The '0' here could signify a base layer or type.
        float rawHeightValue = heightSupplier.get(0, worldX, worldZ, worldSeed);

        // Further processing by the generator...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Seed:** Failing to pass the correct world seed to the `get` method will break world reproducibility. The generator will produce different results each time it is run.
- **Assuming Output Range:** Do not write logic that assumes the returned float is within a specific range like [-1, 1] or [0, 1]. The implementation details are unknown to the consumer. Always query the implementation for its range or normalize the data if required.
- **Stateful Implementations:** Creating an implementation that relies on mutable instance variables without proper synchronization is a critical error. This will lead to severe and difficult-to-debug concurrency bugs during parallel chunk generation.

## Data Pipeline
This component is a source node in many procedural generation data flows. It transforms coordinates into scalar values.

> Flow:
> Terrain Generator -> Requests value for coordinate (x, z) with world seed -> **IFloatCoordinateHashSupplier.get()** -> Returns deterministic float -> Terrain Generator -> Uses float to calculate block height

