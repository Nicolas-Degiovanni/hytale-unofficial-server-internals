---
description: Architectural reference for FloatRangeNoiseSupplier
---

# FloatRangeNoiseSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Transient

## Definition
```java
// Signature
public class FloatRangeNoiseSupplier implements IFloatCoordinateSupplier {
```

## Architecture & Concepts
The FloatRangeNoiseSupplier serves as a critical adapter within the procedural generation framework. Its primary architectural role is to bridge a raw noise function, represented by a NoiseProperty, with a desired output range, represented by an IFloatRange.

This decoupling is essential for creating modular and reusable world generation components. A single, complex NoiseProperty (e.g., for continental elevation) can be adapted by multiple FloatRangeNoiseSupplier instances to produce different outputs, such as biome-specific terrain heights or temperature gradients, without altering the underlying noise algorithm.

It implements the IFloatCoordinateSupplier interface, making it a pluggable component for any system that consumes coordinate-based floating-point data, such as terrain heightmap generators or ore distribution systems.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its public constructor, typically by a higher-level procedural system like a BiomeGenerator or a WorldFeature. The creator is responsible for providing the concrete IFloatRange and NoiseProperty dependencies that define its behavior.
-   **Scope:** The lifetime of a FloatRangeNoiseSupplier is bound to its owner. It is a configuration-specific object, designed to be created once with a specific range and noise profile and then reused for many coordinate lookups.
-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It is eligible for collection once its owning object is dereferenced.

## Internal State & Concurrency
-   **State:** This class is **effectively immutable**. Its core state, the IFloatRange and NoiseProperty, is established at construction and marked as final. It does not modify this state during its lifetime. The class performs no caching; each call to a get method is a direct computation.
-   **Thread Safety:** This class is **conditionally thread-safe**. It contains no internal locks or mutable state. Its safety in a multi-threaded environment, such as a parallel chunk generation system, is entirely dependent on the thread safety of the IFloatRange and NoiseProperty objects provided during construction. Standard engine noise properties and ranges are designed to be thread-safe.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | float | O(N) | Returns a noise value for a 2D coordinate, scaled to the configured range. Complexity N is determined by the underlying NoiseProperty. |
| get(seed, x, y, z) | float | O(N) | Returns a noise value for a 3D coordinate, scaled to the configured range. Complexity N is determined by the underlying NoiseProperty. |

## Integration Patterns

### Standard Usage
This component is designed to be configured once and reused for all lookups matching that configuration.

```java
// 1. Define the desired output range (e.g., terrain height from 60 to 90)
IFloatRange terrainHeightRange = new StaticFloatRange(60.0f, 90.0f);

// 2. Configure the noise function
NoiseProperty elevationNoise = new NoiseProperty(...);

// 3. Create the supplier by combining the range and noise
IFloatCoordinateSupplier heightSupplier = new FloatRangeNoiseSupplier(
    terrainHeightRange,
    elevationNoise
);

// 4. Use the supplier to get values for specific world coordinates
float heightAtPos = heightSupplier.get(worldSeed, 1024.5, 768.2);
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in Loops:** Never create a new FloatRangeNoiseSupplier inside a tight loop, such as iterating over blocks in a chunk. This is highly inefficient and generates significant garbage collector pressure. Create it once and reuse the instance.
-   **Stateful Dependencies:** Providing a non-thread-safe or stateful implementation of IFloatRange or NoiseProperty will break thread safety. This can lead to severe and difficult-to-diagnose concurrency bugs during parallel world generation.

## Data Pipeline
The FloatRangeNoiseSupplier acts as a data source, transforming coordinate inputs into a scaled floating-point value. The internal data flow for a single get call is linear and synchronous.

> Flow:
> (seed, x, y, z) -> **FloatRangeNoiseSupplier.get()** -> NoiseProperty.get() -> Raw Noise Value [-1, 1] -> IFloatRange.getValue() -> Final Scaled Float Value

