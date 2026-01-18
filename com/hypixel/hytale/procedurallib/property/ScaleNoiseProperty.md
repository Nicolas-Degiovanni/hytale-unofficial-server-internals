---
description: Architectural reference for ScaleNoiseProperty
---

# ScaleNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class ScaleNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The ScaleNoiseProperty is a foundational component within the procedural generation library, implementing the **Decorator Pattern**. It acts as a wrapper around another NoiseProperty object, modifying its behavior without altering its fundamental interface.

Its primary function is to transform the input coordinates of a noise query by a given scale factor before passing them to the underlying noise source. This transformation effectively controls the frequency or "zoom level" of the noise.

-   **Scale > 1.0:** Increases the frequency of the noise, making features appear smaller and more detailed. This is analogous to zooming out.
-   **Scale < 1.0:** Decreases the frequency of the noise, making features appear larger and smoother. This is analogous to zooming in.

This class is a critical building block for composing complex procedural functions. By chaining and combining various NoiseProperty decorators, developers can construct sophisticated, multi-layered noise profiles for tasks such as terrain generation, biome placement, and texture synthesis.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor, typically during the assembly of a procedural generation graph. It is almost always created to wrap another, pre-existing NoiseProperty instance.
-   **Scope:** The object's lifetime is bound to its parent container, such as a world generation profile or a biome definition. It is a value-like object with no independent lifecycle.
-   **Destruction:** Managed entirely by the Java garbage collector. It becomes eligible for collection once the object graph that references it is no longer in scope. No explicit cleanup is necessary.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields, including the reference to the wrapped NoiseProperty and the scale factors, are declared final and are set exclusively at construction time. This guarantees that an instance's behavior is constant throughout its lifetime.
-   **Thread Safety:** **Inherently thread-safe**. Its immutable design ensures that a single ScaleNoiseProperty instance can be safely shared and used across multiple threads without locks or other synchronization primitives. This is essential for high-performance, parallelized world generation systems where multiple workers may need to query the same noise profile concurrently.

## API Surface
The public contract is defined by the NoiseProperty interface. The core functionality is exposed through the overloaded *get* methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(N) | Retrieves a 2D noise value. The complexity N is determined by the wrapped NoiseProperty. |
| get(seed, x, y, z) | double | O(N) | Retrieves a 3D noise value. The complexity N is determined by the wrapped NoiseProperty. |

## Integration Patterns

### Standard Usage
The class is used to modify the frequency of a base noise source. It is a key tool for layering noise at different scales (e.g., large-scale continental shapes combined with small-scale surface details).

```java
// 1. Define a base noise source, for example, Simplex noise.
NoiseProperty baseNoise = new SimplexNoiseProperty();

// 2. Wrap the base noise to decrease its frequency, creating larger features.
//    A scale of 0.01 effectively "zooms in" by a factor of 100.
NoiseProperty lowFrequencyNoise = new ScaleNoiseProperty(baseNoise, 0.01);

// 3. Use the decorated property to sample the noise field.
double largeFeatureValue = lowFrequencyNoise.get(worldSeed, 150.0, 250.0);
```

### Anti-Patterns (Do NOT do this)
-   **Zero Scale:** Do not use a scale factor of zero. This will collapse the corresponding input dimension, causing the `get` method to sample the same point in the underlying noise field for any input on that axis, resulting in a constant or linear output.
-   **Redundant Chaining:** Avoid wrapping a ScaleNoiseProperty with another ScaleNoiseProperty. This is computationally wasteful and harms readability.

    ```java
    // AVOID: Inefficient and confusing
    NoiseProperty inefficient = new ScaleNoiseProperty(new ScaleNoiseProperty(base, 2.0), 3.0);

    // PREFERRED: Mathematically equivalent and more performant
    NoiseProperty efficient = new ScaleNoiseProperty(base, 6.0);
    ```

## Data Pipeline
ScaleNoiseProperty acts as a transformation node within a data processing pipeline. It does not generate data itself but rather manipulates the coordinate space for a downstream node.

> Flow:
> Caller -> Input Coordinates (x, y, z) -> **ScaleNoiseProperty** (Coordinate Multiplication) -> Scaled Coordinates (x', y', z') -> Wrapped NoiseProperty -> Final Noise Value

