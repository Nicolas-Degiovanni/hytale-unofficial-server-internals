---
description: Architectural reference for InvertNoiseProperty
---

# InvertNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class InvertNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The InvertNoiseProperty is a foundational component within the procedural content generation (PCG) framework. It embodies the **Decorator** design pattern, acting as a stateless wrapper that modifies the output of another NoiseProperty instance without altering its underlying implementation.

Its sole function is to receive the output of a wrapped noise function, which is typically in the range of 0.0 to 1.0, and return its inverse (1.0 - value). This simple transformation is a powerful primitive in world generation for flipping topographical features. For example, it can be used to turn a noise map that defines mountains into one that defines valleys, or to convert a river channel into a ridge.

This class is designed for composition. Developers can chain multiple NoiseProperty decorators together to build complex, layered noise profiles from a set of simple, reusable components.

### Lifecycle & Ownership
- **Creation:** Instantiated directly by procedural generation logic when constructing a noise graph. It is not managed by a service locator or dependency injection framework.
- **Scope:** The lifetime of an InvertNoiseProperty instance is strictly bound to the lifecycle of the noise graph that contains it, such as a biome definition or a specific world generation pass. It is a short-lived object.
- **Destruction:** The object is eligible for garbage collection as soon as its parent noise graph is no longer referenced. No explicit destruction or resource cleanup is necessary.

## Internal State & Concurrency
- **State:** This class is **immutable**. Its single internal field, noiseProperty, is marked as final and is assigned only during construction. It maintains no other internal state.
- **Thread Safety:** This class is conditionally thread-safe. Its safety is entirely dependent on the thread safety of the wrapped NoiseProperty instance provided during construction. The InvertNoiseProperty class itself introduces no locks, synchronization, or mutable state, making it a transparent pass-through regarding concurrency.

**WARNING:** If a non-thread-safe NoiseProperty is passed to the constructor, all calls to this InvertNoiseProperty instance must be externally synchronized.

## API Surface
The public contract is defined by the NoiseProperty interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(T) | Calculates the 2D noise value from the wrapped property and returns its inverse. T is the complexity of the wrapped call. |
| get(seed, x, y, z) | double | O(T) | Calculates the 3D noise value from the wrapped property and returns its inverse. T is the complexity of the wrapped call. |

## Integration Patterns

### Standard Usage
The intended use is to wrap an existing NoiseProperty to invert its output as part of a larger noise generation pipeline.

```java
// 1. Define a base noise function, for example, Simplex noise.
NoiseProperty baseNoise = new SimplexNoiseProperty();

// 2. Wrap the base noise to invert its output.
NoiseProperty invertedNoise = new InvertNoiseProperty(baseNoise);

// 3. Use the inverted noise in the world generator.
//    Where baseNoise would produce a peak (e.g., 0.9), invertedNoise will produce a trough (0.1).
double value = invertedNoise.get(worldSeed, 128.5, 70.2);
```

### Anti-Patterns (Do NOT do this)
- **Null Wrapping:** Never construct this class with a null NoiseProperty. This will lead to a `NullPointerException` during the `get` call and is an unrecoverable state.
- **Redundant Chaining:** Avoid chaining two instances of InvertNoiseProperty. This is computationally wasteful and logically incorrect, as it inverts the value twice, effectively returning the original value.
  ```java
  // AVOID: This is equivalent to just using baseNoise but with extra overhead.
  NoiseProperty uselessWrapper = new InvertNoiseProperty(new InvertNoiseProperty(baseNoise));
  ```

## Data Pipeline
The data flow for this component is a simple, linear transformation. It acts as a single stage in a potentially much larger noise processing pipeline.

> Flow:
> World Generator Request -> **InvertNoiseProperty.get(params)** -> Wrapped NoiseProperty.get(params) -> `return 1.0 - result` -> Transformed Noise Value

