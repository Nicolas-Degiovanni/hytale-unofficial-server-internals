---
description: Architectural reference for CurveNoiseProperty
---

# CurveNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class CurveNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The CurveNoiseProperty is a decorator component within the procedural generation framework. It acts as a wrapper around an existing NoiseProperty implementation, modifying its output by applying a mathematical function. This class is a fundamental tool for shaping and controlling the characteristics of generated noise, which is essential for creating aesthetically pleasing and varied terrain.

Its primary role is to remap the distribution of noise values. For example, a standard noise function might produce a linear distribution of values between -1.0 and 1.0. By wrapping it with a CurveNoiseProperty, a developer can apply a power curve, a sine wave, or any other function to create more contrast, flatten certain value ranges, or introduce non-linear features.

This class adheres to the **Decorator Pattern**. It implements the NoiseProperty interface, allowing it to be seamlessly inserted into a chain of other noise modifiers and processors without altering the calling code. This composability is key to building complex and layered procedural generation pipelines.

The public nested class, PowerCurve, is a provided implementation of a common remapping function used to control the steepness and curvature of terrain features.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor: `new CurveNoiseProperty(noise, function)`. This is typically done during the setup phase of a world generator, where a complete noise generation pipeline is assembled from various components.
- **Scope:** The object's lifetime is bound to the parent object that configures the noise pipeline, such as a Biome or WorldGenerator configuration. It is not a global or session-scoped service.
- **Destruction:** The object holds no native resources and is managed entirely by the Java garbage collector. It becomes eligible for collection once the world generation process that uses it is complete and its configuration is no longer referenced.

## Internal State & Concurrency
- **State:** The internal state consists of two final fields: the wrapped NoiseProperty and the DoubleUnaryOperator function. This state is set at construction and cannot be changed, making the CurveNoiseProperty instance **immutable**.
- **Thread Safety:** This class is inherently **thread-safe**. Its methods do not modify internal state and their output is a pure function of the input arguments and the immutable configuration. Concurrency safety of a full pipeline depends on the thread safety of the wrapped NoiseProperty and the provided function. The included PowerCurve class is also immutable and thread-safe, making it safe for use in multi-threaded world generation contexts.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CurveNoiseProperty(noise, function) | constructor | O(1) | Constructs a noise modifier that wraps an existing noise source. |
| get(seed, x, y) | double | O(N) | Retrieves a 2D noise value from the wrapped source, applies the curve function, and returns the result. Complexity is inherited from the wrapped source. |
| get(seed, x, y, z) | double | O(N) | Retrieves a 3D noise value from the wrapped source, applies the curve function, and returns the result. Complexity is inherited from the wrapped source. |
| PowerCurve(a, b) | constructor | O(1) | Constructs a pre-defined power curve function. This is a public nested utility class. |

## Integration Patterns

### Standard Usage
The primary pattern is to chain this decorator onto another NoiseProperty to reshape its output. This is often done when defining the noise layers for a specific biome or terrain feature.

```java
// Assume 'baseNoise' is an existing NoiseProperty, like SimplexNoiseProperty
NoiseProperty baseNoise = ...;

// Create a power curve to make flat areas flatter and steep areas steeper
DoubleUnaryOperator curve = new CurveNoiseProperty.PowerCurve(2.5, 0.5);

// Wrap the base noise with the curve modifier
NoiseProperty shapedNoise = new CurveNoiseProperty(baseNoise, curve);

// Use the modified noise property in the generator
double value = shapedNoise.get(seed, x, y);
```

### Anti-Patterns (Do NOT do this)
- **Expensive Functions:** Avoid providing a computationally expensive DoubleUnaryOperator. The function is executed for every single noise sample, and any inefficiency will have a massive performance impact on world generation.
- **Unbounded Inputs:** The provided PowerCurve class assumes an input operand between 0.0 and 1.0. While the implementation handles this by inverting the value, feeding it noise outside the standard [-1.0, 1.0] range may produce unexpected or mathematically unstable results. Always normalize noise before applying curves unless the curve is specifically designed for a wider range.

## Data Pipeline
CurveNoiseProperty acts as a transformation step in the noise generation data flow. It does not generate data but rather reshapes it.

> Flow:
> (Coordinates, Seed) -> Underlying NoiseProperty -> Raw Noise Value -> **CurveNoiseProperty** -> Transformed Noise Value -> World Generator / Voxel Engine

