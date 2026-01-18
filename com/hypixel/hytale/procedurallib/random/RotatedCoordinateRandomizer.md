---
description: Architectural reference for RotatedCoordinateRandomizer
---

# RotatedCoordinateRandomizer

**Package:** com.hypixel.hytale.procedurallib.random
**Type:** Transient

## Definition
```java
// Signature
public class RotatedCoordinateRandomizer implements ICoordinateRandomizer {
```

## Architecture & Concepts
The RotatedCoordinateRandomizer is an implementation of the **Decorator** design pattern. It does not generate random values itself. Instead, it wraps an existing ICoordinateRandomizer instance and modifies its behavior by transforming the input coordinates before delegation.

Its primary role within the procedural generation engine is to apply a spatial rotation to a coordinate system before sampling a noise function or random value. This is fundamental for creating features that are not aligned with the world's primary X, Y, and Z axes, such as diagonal mountain ranges, angled cave systems, or rotated biome patterns.

This class cleanly separates the concern of coordinate transformation from the concern of random number generation, allowing for flexible composition of procedural generation behaviors.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, typically by a higher-level system responsible for composing procedural generation pipelines, such as a WorldGenerator or BiomeFeaturePlacer. It requires a concrete ICoordinateRandomizer and a CoordinateRotator to be provided at construction.
- **Scope:** The object's lifetime is transient and bound to the specific generation task that created it. It is not a long-lived service and is typically discarded after the relevant world chunk or feature has been generated.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It becomes eligible for collection as soon as the owning generator that created it goes out of scope.

## Internal State & Concurrency
- **State:** The internal state of this class is **immutable**. The wrapped *randomizer* and *rotation* objects are assigned via final fields in the constructor and cannot be changed during the object's lifetime.
- **Thread Safety:** This class is conditionally thread-safe. It introduces no synchronization or mutable state of its own. Its thread safety is entirely dependent on the thread safety of the ICoordinateRandomizer and CoordinateRotator objects provided during its construction. If the wrapped objects are thread-safe, this class can be safely used across multiple threads.

**WARNING:** The caller is responsible for ensuring that the injected dependencies are appropriate for the intended concurrent environment.

## API Surface
The public API consists of methods that override the ICoordinateRandomizer interface. All methods follow the same internal pattern: rotate coordinates, then delegate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| randomDoubleX(seed, x, y) | double | O(1) | Rotates the 2D coordinate (x, y), then calls the wrapped randomizer. |
| randomDoubleY(seed, x, y) | double | O(1) | Rotates the 2D coordinate (x, y), then calls the wrapped randomizer. |
| randomDoubleX(seed, x, y, z) | double | O(1) | Rotates the 3D coordinate (x, y, z), then calls the wrapped randomizer. |
| randomDoubleY(seed, x, y, z) | double | O(1) | Rotates the 3D coordinate (x, y, z), then calls the wrapped randomizer. |
| randomDoubleZ(seed, x, y, z) | double | O(1) | Rotates the 3D coordinate (x, y, z), then calls the wrapped randomizer. |

## Integration Patterns

### Standard Usage
This class is intended to be composed with other randomizers to create more complex procedural effects. The typical use case is to wrap a base noise function, like Perlin noise, to get a rotated version of that noise.

```java
// Example: Creating a 45-degree rotated Perlin noise generator
ICoordinateRandomizer baseNoise = new PerlinNoiseRandomizer();
CoordinateRotator rotation = new CoordinateRotator(45.0); // Assume this exists

// Wrap the base noise function with the rotator
ICoordinateRandomizer rotatedNoise = new RotatedCoordinateRandomizer(baseNoise, rotation);

// Now, sampling rotatedNoise at (10, 0) is equivalent to sampling
// baseNoise at a rotated coordinate.
double value = rotatedNoise.randomDoubleX(worldSeed, 10.0, 0.0);
```

### Anti-Patterns (Do NOT do this)
- **Wrapping Non-Coordinate-Based Randomizers:** Do not wrap a randomizer that ignores its coordinate inputs. The rotation would be a pointless and wasteful computation.
- **Excessive Chaining:** While decorators can be chained, chaining multiple RotatedCoordinateRandomizer instances can lead to confusing behavior and performance degradation. It is architecturally cleaner to compose rotations within a single CoordinateRotator instance if possible.

## Data Pipeline
This component acts as a pure transformation step within a larger data flow. It does not source or sink data but rather modifies it in-flight.

> Flow:
> Input Coordinates (x, y, z) -> **RotatedCoordinateRandomizer** [applies CoordinateRotator] -> Transformed Coordinates (px, py, pz) -> Wrapped ICoordinateRandomizer -> Output (double)

