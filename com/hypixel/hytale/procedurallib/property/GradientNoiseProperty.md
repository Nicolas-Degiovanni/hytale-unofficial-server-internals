---
description: Architectural reference for GradientNoiseProperty
---

# GradientNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class GradientNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The GradientNoiseProperty class is a decorator within the procedural generation library. It wraps an existing NoiseProperty implementation to transform its output. Instead of returning the raw noise value (often representing height), this class calculates the local *gradient* of the noise field at a given point.

The gradient is a vector that points in the direction of the steepest ascent of the noise function. This class approximates this vector by sampling the underlying noise at three points in a small right-angled triangle: `(x, y)`, `(x + distance, y)`, and `(x, y + distance)`. The differences between these samples yield `dx` and `dy` components, representing the rate of change along each axis.

Based on the configured GradientMode, it can return one of three interpretations of this gradient:
*   **MAGNITUDE:** The length of the gradient vector. This corresponds to the steepness or slope of the noise field. High values indicate cliffs or sharp changes, while low values indicate flat areas.
*   **ANGLE:** The direction of the gradient vector, mapped to a circular range. This is useful for determining flow direction, such as for rivers or erosion patterns.
*   **ANGLE_ABS:** The absolute angle, useful for identifying features that are direction-agnostic but slope-dependent.

This component is a fundamental building block for moving beyond simple heightmap generation and into more complex procedural analysis, such as hydrology simulation or biome placement based on terrain slope.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its public constructor, typically during the setup phase of a procedural world generator. It is a configuration object that composes a more complex noise pipeline.
- **Scope:** The object's lifetime is tied to the parent noise graph or generator that created it. It is stateless and designed to be short-lived, existing only for the duration of a specific generation task.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It is reclaimed once the reference from its parent generator is dropped.

## Internal State & Concurrency
- **State:** **Immutable.** All internal fields, including the wrapped NoiseProperty, mode, and distance, are final and set at construction time. The class holds no mutable state.
- **Thread Safety:** **Inherently Thread-Safe.** Due to its immutable nature and the absence of side effects in its computation methods, an instance of GradientNoiseProperty can be safely shared and accessed by multiple threads simultaneously without external synchronization. This is critical for parallelizing world generation chunks.

## API Surface
The public contract is defined by the NoiseProperty interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(1) | Calculates the gradient-derived property at a 2D coordinate. This operation is computationally cheap but involves three lookups into the underlying noise source. |
| get(seed, x, y, z) | double | N/A | **Not implemented.** Throws UnsupportedOperationException. This class is strictly designed for 2D gradient analysis. |

## Integration Patterns

### Standard Usage
GradientNoiseProperty is used to decorate another noise source to extract slope or direction information.

```java
// Assume 'baseNoise' is an existing NoiseProperty, like SimplexNoiseProperty
NoiseProperty baseNoise = new SimplexNoiseProperty(...);

// Create a property that returns the steepness of the base noise
NoiseProperty steepness = new GradientNoiseProperty(baseNoise, GradientMode.MAGNITUDE, 1.0, 1.0);

// Create a property that returns the direction of the slope
NoiseProperty flowDirection = new GradientNoiseProperty(baseNoise, GradientMode.ANGLE, 1.0, 1.0);

// Use the new property in a generator
double terrainSteepness = steepness.get(seed, x, y);
if (terrainSteepness > 0.8) {
    // Place cliff face
}
```

### Anti-Patterns (Do NOT do this)
- **3D Invocation:** Do not call the `get(seed, x, y, z)` method. It will throw an UnsupportedOperationException and crash the generation thread. This implementation is strictly two-dimensional.
- **Zero Distance or Normalize:** Constructing this class with a `distance` or `normalize` value of zero is a critical configuration error. It will lead to division-by-zero errors during the `invNormalize` calculation or produce meaningless gradient vectors where `dx` and `dy` are always zero.
- **Wrapping High-Frequency Noise:** Wrapping a very high-frequency base noise source can produce chaotic and unusable gradient information. The `distance` parameter should be tuned relative to the feature size of the underlying noise.

## Data Pipeline
The class functions as a pure transformation stage within a larger data processing pipeline.

> Flow:
> (seed, x, y) -> **GradientNoiseProperty.get()** -> [3x call to underlying NoiseProperty.get()] -> [Calculate dx, dy vector] -> [Process vector via ANGLE or MAGNITUDE logic] -> double [0.0, 1.0]

