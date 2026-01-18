---
description: Architectural reference for DistortedNoiseProperty
---

# DistortedNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class DistortedNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The DistortedNoiseProperty is a fundamental component within the procedural generation framework, designed to create more complex and natural-looking noise patterns. It implements the **Decorator** design pattern, wrapping an existing NoiseProperty to alter its behavior without changing its public interface.

Its core function is to apply a transformation, or "distortion", to the input coordinates *before* they are evaluated by the underlying noise function. This technique, often called domain warping, is critical for breaking up the uniform grid-like appearance of standard noise algorithms like Perlin or Simplex. By warping the input space, this class can produce organic effects such as turbulence, swirls, and other non-linear formations essential for generating realistic terrain, textures, and biome distributions.

This class is a compositional building block. It is designed to be chained and combined with other NoiseProperty implementations, allowing developers to construct highly sophisticated and layered noise generators from simpler primitives.

## Lifecycle & Ownership
- **Creation:** An instance is created during the configuration phase of a procedural generation task, typically by a world generator builder or a configuration parser. It is constructed with its required dependencies: a base NoiseProperty to be decorated and an ICoordinateRandomizer to define the distortion.
- **Scope:** The object's lifetime is bound to the specific procedural generation configuration it belongs to. It is generally a short-lived object, existing only for the duration of a single, discrete generation job, such as creating a world chunk.
- **Destruction:** The object is managed by the Java garbage collector and is reclaimed once the procedural configuration that holds a reference to it goes out of scope. No explicit destruction or resource cleanup is necessary.

## Internal State & Concurrency
- **State:** **Immutable**. The internal fields for the wrapped NoiseProperty and ICoordinateRandomizer are declared final and are injected exclusively through the constructor. The class itself maintains no mutable state. All method calls are pure functions whose results depend solely on the input arguments and the immutable state of its dependencies.
- **Thread Safety:** **Conditionally Thread-Safe**. The thread safety of a DistortedNoiseProperty instance is entirely inherited from its dependencies. If the provided NoiseProperty and ICoordinateRandomizer objects are themselves thread-safe, then this class can be safely accessed from multiple threads. Given its role in parallelized tasks like world generation, its dependencies are generally expected to be thread-safe.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(C) | Samples the noise value at a 2D coordinate. The coordinate is first transformed by the internal randomizer before being passed to the wrapped noise property. |
| get(seed, x, y, z) | double | O(C) | Samples the noise value at a 3D coordinate. The coordinate is first transformed by the internal randomizer before being passed to the wrapped noise property. |

## Integration Patterns

### Standard Usage
The class is used to layer a distortion effect on top of a base noise function. This is a common pattern for increasing visual complexity.

```java
// 1. Define a base noise function (e.g., Perlin)
NoiseProperty baseNoise = new PerlinNoiseProperty(/*...params...*/);

// 2. Define a distortion function, which is often another noise source
//    configured as a coordinate randomizer.
ICoordinateRandomizer distortion = new NoiseCoordinateRandomizer(new SimplexNoiseProperty(/*...params...*/), 16.0);

// 3. Decorate the base noise with the distortion
NoiseProperty finalNoise = new DistortedNoiseProperty(baseNoise, distortion);

// 4. Use the final, complex noise property in the generator
double value = finalNoise.get(worldSeed, 123.4, 567.8);
```

### Anti-Patterns (Do NOT do this)
- **Null Dependencies:** Never construct a DistortedNoiseProperty with null arguments. The constructor does not perform null-safety checks, and doing so will inevitably lead to a NullPointerException when the `get` method is invoked.
- **Excessive Distortion:** Applying a distortion with an extremely high amplitude can warp the input coordinates so severely that it produces visual artifacts, aliasing, or loss of detail from the underlying noise. The distortion effect must be carefully tuned.
- **Recursive Decoration:** While technically possible, creating a chain where a DistortedNoiseProperty uses a randomizer that depends on the same noise instance can lead to infinite recursion and a StackOverflowError.

## Data Pipeline
The primary role of this class is to intercept and transform coordinate data before it reaches a base noise function.

> Flow:
> Input Coordinates (x, y, z) & Seed -> **DistortedNoiseProperty.get()** -> ICoordinateRandomizer -> Distorted Coordinates (x', y', z') -> Wrapped NoiseProperty.get() -> Final Noise Value

