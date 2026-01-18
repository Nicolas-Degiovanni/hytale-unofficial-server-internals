---
description: Architectural reference for OffsetNoiseProperty
---

# OffsetNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient Decorator

## Definition
```java
// Signature
public class OffsetNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The OffsetNoiseProperty is a fundamental component within the procedural generation library, implementing the **Decorator** design pattern. Its sole purpose is to wrap an existing NoiseProperty implementation and apply a fixed spatial translation to the input coordinates before they are passed to the underlying noise function.

This class acts as a coordinate transformation layer in a procedural generation pipeline. It allows a single, complex noise definition (the wrapped NoiseProperty) to be sampled at different locations, effectively creating unique-looking but structurally similar outputs. This is a critical technique for generating varied world features, such as biomes or ore veins, from a common set of base noise functions without redefining them. By simply wrapping a noise source with an OffsetNoiseProperty, developers can "move" the generated feature in the world space.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor, typically during the assembly of a procedural generation graph. It is not managed by a dependency injection container or service locator. Higher-level systems, like a WorldGenerator or BiomeBuilder, are responsible for composing these objects.
- **Scope:** The lifetime of an OffsetNoiseProperty is tied to its owning configuration object. It is a value-like object that exists only as part of a larger, temporary generation task. It does not persist across sessions or server restarts.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection as soon as the procedural graph it belongs to is no longer in scope. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields, including the reference to the wrapped NoiseProperty and the offset values, are declared final and are set exclusively at construction time. The object's state cannot be modified after instantiation.
- **Thread Safety:** **Conditionally Thread-Safe**. This class contains no mutable state or synchronization primitives. Its thread safety is therefore entirely dependent on the thread safety of the wrapped NoiseProperty instance it contains. If the decorated NoiseProperty is thread-safe, then any instance of OffsetNoiseProperty wrapping it is also safe for concurrent use.

    **Warning:** Passing a non-thread-safe NoiseProperty to the constructor will result in a non-thread-safe OffsetNoiseProperty. The immutability of this wrapper provides no protection for the object it decorates.

## API Surface
The public API is minimal, focusing entirely on the NoiseProperty contract. Getters for internal state are provided but are not central to its function.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(T) | Samples the wrapped 2D noise function at the translated coordinates (x + offsetX, y + offsetY). Complexity is determined by the wrapped noise property. |
| get(seed, x, y, z) | double | O(T) | Samples the wrapped 3D noise function at the translated coordinates (x + offsetX, y + offsetY, z + offsetZ). Complexity is determined by the wrapped noise property. |

## Integration Patterns

### Standard Usage
The primary use case is to wrap a base noise function to shift its output in world space. This is a common pattern in procedural graph assembly.

```java
// 1. Define a base noise source, for example, a simplex noise generator.
NoiseProperty baseNoise = new SimplexNoiseProperty();

// 2. Wrap the base noise with an offset to create a translated view.
//    This is useful for generating a similar feature at a different location.
NoiseProperty translatedNoise = new OffsetNoiseProperty(baseNoise, 1024.0, 0.0, -512.0);

// 3. Sample the noise.
//    This call is functionally equivalent to baseNoise.get(seed, 1034.0, 15.0, -502.0)
double value = translatedNoise.get(worldSeed, 10.0, 15.0, 10.0);
```

### Anti-Patterns (Do NOT do this)
- **Redundant Nesting:** Avoid wrapping an OffsetNoiseProperty with another OffsetNoiseProperty. This is computationally wasteful and harms readability.

    ```java
    // BAD: Creates two object allocations and an extra method call per sample.
    NoiseProperty base = new SimplexNoiseProperty();
    NoiseProperty offset1 = new OffsetNoiseProperty(base, 100.0, 0.0, 0.0);
    NoiseProperty offset2 = new OffsetNoiseProperty(offset1, 50.0, 0.0, 0.0);

    // GOOD: Pre-calculate the final offset and create a single wrapper.
    NoiseProperty combinedOffset = new OffsetNoiseProperty(base, 150.0, 0.0, 0.0);
    ```
- **Wrapping Stateful Objects:** Do not wrap a NoiseProperty that maintains internal, mutable state if the resulting object is intended for concurrent use. The OffsetNoiseProperty provides no synchronization.

## Data Pipeline
OffsetNoiseProperty is a pure transformation node within a data flow. It does not source or sink data but rather modifies it in-flight.

> Flow:
> Input Coordinates (x, y, z) -> **OffsetNoiseProperty** (adds internal offsets) -> Translated Coordinates (x', y', z') -> Wrapped NoiseProperty -> Final Noise Value

