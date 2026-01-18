---
description: Architectural reference for FloatRange
---

# FloatRange

**Package:** com.hypixel.hytale.math.range
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class FloatRange {
```

## Architecture & Concepts
The FloatRange class is a fundamental mathematical primitive used throughout the engine to represent a continuous, inclusive range of floating-point values. Its primary function is to encapsulate a minimum and maximum bound, providing utility methods for interpolation and containment checks.

This class is not a service or a manager; it is a simple data-holding object, often referred to as a Plain Old Java Object (POJO). Its architectural significance lies in its widespread use for configuring gameplay mechanics where variability is required. Examples include defining the potential damage range of a weapon, the lifetime variance of a particle effect, or the randomized pitch of a sound effect.

The inclusion of a static CODEC field, FloatRangeArrayCodec, is a critical design choice. It signals that FloatRange instances are intended to be serialized and deserialized as part of larger data models, such as entity definitions or world generation configurations loaded from disk.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand, typically through direct instantiation with `new FloatRange(min, max)`. They are also created implicitly by the data loading systems when deserializing game assets that utilize the associated CODEC.
- **Scope:** The lifetime of a FloatRange object is strictly tied to its owner. It is a transient, value-like object that persists only as long as the containing object (e.g., a weapon configuration, an animation definition) is held in memory.
- **Destruction:** Instances are managed by the Java garbage collector. They are eligible for cleanup as soon as they are no longer referenced. No explicit destruction or cleanup methods are provided or required.

## Internal State & Concurrency
- **State:** FloatRange is a **mutable** object. The `setInclusiveMin` and `setInclusiveMax` methods directly modify its internal state. It maintains a cached `range` field which is automatically re-calculated whenever the bounds are changed. This is a micro-optimization to avoid repeated calculations in high-frequency calls to `getFloat`.

- **Thread Safety:** This class is **not thread-safe**. Concurrent calls to setters (e.g., `setInclusiveMin`) and accessors (e.g., `getFloat`) from different threads can lead to unpredictable behavior and incorrect calculations. The read/write operations on its fields are not atomic.

    **WARNING:** If a FloatRange instance must be shared between threads, all access to it must be synchronized externally using locks or other concurrency control mechanisms. Failure to do so will result in race conditions.

## API Surface
The public API provides the core functionality for defining, mutating, and querying the numerical range.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FloatRange(min, max) | constructor | O(1) | Creates a new range with the specified inclusive bounds. |
| setInclusiveMin(min) | void | O(1) | Mutates the lower bound of the range. |
| setInclusiveMax(max) | void | O(1) | Mutates the upper bound of the range. |
| getFloat(factor) | float | O(1) | Interpolates a value within the range. The factor is expected to be between 0.0 and 1.0. |
| includes(value) | boolean | O(1) | Checks if the given value falls within the min/max bounds. |
| CODEC | FloatRangeArrayCodec | O(1) | Static final field providing the serialization/deserialization handler. |

## Integration Patterns

### Standard Usage
FloatRange is most commonly used to configure and then query a variable value. A system will typically hold a FloatRange instance as a configuration parameter and use `getFloat` with a random factor to produce varied results.

```java
// Example: A particle emitter using FloatRange to vary particle lifetime.
// This configuration would be loaded from a data file.
FloatRange lifetimeRange = new FloatRange(1.5f, 3.0f);

// In the game loop, when a new particle is spawned:
float randomFactor = Random.nextFloat(); // Produces a value between 0.0 and 1.0
float particleLifetime = lifetimeRange.getFloat(randomFactor);
particle.setLifetime(particleLifetime);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not modify a FloatRange instance from one thread while it is being read by another. This is the most common source of errors when using this class.

    ```java
    // ANTI-PATTERN: Shared mutable state without synchronization
    FloatRange sharedRange = new FloatRange(0, 10);

    // Thread A
    new Thread(() -> {
        while(true) { sharedRange.setInclusiveMax(newValue); }
    }).start();

    // Thread B
    new Thread(() -> {
        while(true) { System.out.println(sharedRange.getFloat(0.5f)); } // Unpredictable results
    }).start();
    ```

- **Misinterpreting Interpolation Logic:** The internal `range` calculation is `inclusiveMax - inclusiveMin + 1.0F`. This is an atypical formula for floating-point ranges and may be a remnant of integer-based logic. While the `getFloat` method clamps the final value correctly, developers should not attempt to replicate the interpolation logic externally, as it may produce unexpected off-by-one style errors if the clamping is forgotten. Rely solely on the `getFloat` method.

## Data Pipeline
FloatRange is not a processing stage in a pipeline but rather a data payload that flows through one. Its most frequent role is being deserialized from configuration files into active game objects.

> Flow:
> Game Asset File (e.g., JSON) -> Hytale Asset Loader -> **FloatRangeArrayCodec** -> **FloatRange Instance** -> Game System (e.g., Particle Emitter)

