---
description: Architectural reference for IFloatRange
---

# IFloatRange

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IFloatRange {
```

## Architecture & Concepts
The IFloatRange interface is a fundamental contract within the procedural generation library. It establishes a standardized abstraction for any system that supplies a floating-point value based on a given context. Its primary role is to decouple procedural algorithms, such as terrain heightmap generation or feature placement, from the specific method used to generate a value.

This interface allows a single procedural system to operate on a wide variety of inputs without modification. For example, a system that determines tree height can be configured to use an IFloatRange implementation that returns a constant, a value from a simple random distribution, or a complex value derived from a 3D noise function like Perlin or Simplex noise.

The various overloads of the *getValue* method are the core of this design. They provide a context-sensitive mechanism for value resolution, allowing the same IFloatRange instance to be queried with different inputs—such as a simple randomizer, a 2D world coordinate, or a 3D world coordinate—to produce a deterministic or stochastic result.

## Lifecycle & Ownership
As an interface, IFloatRange itself has no lifecycle. The following describes the expected lifecycle of its concrete implementations.

- **Creation:** Implementations are typically instantiated and configured during the initialization phase of a procedural generation pipeline. They are often defined in data files (e.g., JSON or YAML) and deserialized by a configuration loader, which then injects them into the relevant procedural generators.
- **Scope:** The lifetime of an IFloatRange implementation is tied to the scope of the generation task it serves. For world generation, an instance may persist for the entire duration of the chunk or region generation process.
- **Destruction:** Instances are eligible for garbage collection once the procedural generation task is complete and all references from the generator are released. There is no explicit destruction or cleanup method defined in the contract.

## Internal State & Concurrency
The contract makes no guarantees about the internal state or thread safety of its implementations. Consumers of this interface must be aware of the nature of the specific implementation they are interacting with.

- **State:** Implementations can range from entirely stateless (e.g., a class that always returns a constant value) to highly stateful (e.g., a noise function that caches results for specific coordinates).
- **Thread Safety:** Implementations are **not guaranteed to be thread-safe**. A common pattern is for a generator to use a single-threaded `Random` instance, which is then passed into the `getValue(Random var1)` method. Sharing such an implementation across multiple threads without external synchronization will lead to race conditions and non-deterministic output. It is the responsibility of the calling system to manage concurrency.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue(float) | float | O(1) | Resolves a value using a single float as input, often for simple interpolation. |
| getValue(FloatSupplier) | float | O(1) | Resolves a value using a supplier, allowing for deferred or chained value resolution. |
| getValue(Random) | float | O(1) | Resolves a value using a provided Random instance for stochastic results. |
| getValue(int, double, double, IDoubleCoordinateSupplier2d) | float | Implementation-dependent | Resolves a value based on a 2D coordinate context, typically for terrain or biome maps. |
| getValue(int, double, double, double, IDoubleCoordinateSupplier3d) | float | Implementation-dependent | Resolves a value based on a 3D coordinate context, essential for volumetric noise and 3D feature placement. |

## Integration Patterns

### Standard Usage
An IFloatRange implementation is typically retrieved from a configuration asset and passed to a procedural algorithm. The algorithm then calls the appropriate `getValue` method, providing its current operational context (e.g., world coordinates) to get a result.

```java
// A procedural generator receives an IFloatRange for determining feature scale
public void placeFeature(WorldContext ctx, IFloatRange scaleSupplier) {
    // Get the scale value based on the feature's 3D world position
    float scale = scaleSupplier.getValue(
        ctx.getSeed(),
        ctx.getPosition().x,
        ctx.getPosition().y,
        ctx.getPosition().z,
        // Assuming a default coordinate supplier
        IDoubleCoordinateSupplier3d.IDENTITY
    );

    // Use the resolved scale value
    createFeatureAt(ctx.getPosition(), scale);
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming Statelessness:** Do not assume that calling `getValue` with the same parameters will always yield the same result. Some implementations are stochastic and depend on the internal state of the `Random` object passed to them.
- **Ignoring Context:** Do not use a generic overload like `getValue(new Random())` when a more specific, coordinate-based context is available. Doing so defeats the purpose of deterministic procedural generation and will result in non-reproducible worlds.
- **Incorrect Overload:** Calling the 2D `getValue` method for a 3D context (or vice versa) will produce incorrect or unexpected results, often manifesting as visual artifacts like sheared terrain or repeating patterns on the Y-axis.

## Data Pipeline
The IFloatRange interface acts as a source node in a procedural generation data flow. It does not typically process incoming data but rather originates it based on contextual input.

> Flow:
> Procedural Generator (e.g., BiomePlacer) -> Provides Context (Seed, X, Y, Z) -> **IFloatRange Implementation** -> Computes Value (e.g., via Noise Function) -> Returns float -> Generator uses float for logic (e.g., set biome type)

