---
description: Architectural reference for RotateNoiseProperty
---

# RotateNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient / Component

## Definition
```java
// Signature
public class RotateNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The RotateNoiseProperty is a fundamental component within the procedural generation library, implementing the **Decorator** design pattern. Its primary architectural role is to act as a wrapper around another NoiseProperty implementation, transforming the input coordinate space before the underlying noise function is sampled.

This class decouples the core noise algorithm (e.g., Simplex, Perlin) from its spatial orientation. By intercepting the coordinate inputs (x, y, z), applying a rotation via a CoordinateRotator, and then passing the transformed coordinates to the wrapped noise source, it allows for the creation of procedural features that are not aligned with the world's primary axes. This is essential for generating more natural and varied environmental features, such as angled mountain ranges or swirling patterns.

It is a building block, intended to be composed with other NoiseProperty decorators to construct a complex procedural generation graph.

### Lifecycle & Ownership
-   **Creation:** Instances are created directly via the public constructor (`new RotateNoiseProperty(...)`). This is typically done within higher-level procedural configuration code or a graph builder that assembles the final noise function. It is not managed by a service locator or dependency injection framework.
-   **Scope:** The lifetime of a RotateNoiseProperty instance is bound to the parent object that holds a reference to it, usually a composite procedural function or a world generation profile. It is a value-like object with no independent lifecycle.
-   **Destruction:** The object is eligible for garbage collection as soon as its parent configuration is no longer referenced. It requires no explicit cleanup or resource management.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal fields for the wrapped noise property and the rotator are marked `final` and are assigned exclusively at construction time. The class holds no mutable state and performs no internal caching. Each call to a `get` method is a pure function of its inputs.
-   **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a single RotateNoiseProperty instance can be safely shared and accessed by multiple threads concurrently without any need for external synchronization. This is a critical property for enabling parallelized chunk and world generation.

## API Surface
The public contract is defined by the NoiseProperty interface, focusing exclusively on sampling the noise field.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(1) | Transforms the 2D coordinates and returns the value from the wrapped noise source at the new position. |
| get(seed, x, y, z) | double | O(1) | Transforms the 3D coordinates and returns the value from the wrapped noise source at the new position. |

## Integration Patterns

### Standard Usage
The intended use is to chain this property onto an existing noise source to introduce rotation. This is a common step in building a procedural function stack.

```java
// 1. Define a base noise source to be decorated
NoiseProperty baseNoise = new SimplexNoiseProperty(frequency, amplitude);

// 2. Define the rotation transformation
CoordinateRotator yAxisRotator = new CoordinateRotator(45.0, Axis.Y);

// 3. Create the decorator, wrapping the base noise
NoiseProperty rotatedNoise = new RotateNoiseProperty(baseNoise, yAxisRotator);

// 4. Use the composed property to sample the rotated noise field
double value = rotatedNoise.get(worldSeed, 100.5, 50.0);
```

### Anti-Patterns (Do NOT do this)
-   **Null Dependencies:** Do not pass `null` for the `noise` or `rotation` constructor arguments. The class performs no internal null-safety checks and will deterministically throw a NullPointerException during any subsequent `get` call.
-   **Stateful Dependencies:** Wrapping a non-thread-safe or stateful NoiseProperty implementation with this class does not make the composite object thread-safe. The thread-safety guarantee of RotateNoiseProperty relies on its dependencies also being thread-safe.

## Data Pipeline
RotateNoiseProperty acts as a pure transformation step within a larger data flow. It does not initiate or terminate a pipeline but rather modifies data in-flight.

> Flow:
> Input Coordinates -> **RotateNoiseProperty** (applies CoordinateRotator) -> Transformed Coordinates -> Wrapped NoiseProperty -> Output Noise Value

