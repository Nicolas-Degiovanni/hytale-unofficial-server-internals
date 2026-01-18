---
description: Architectural reference for FloatRange
---

# FloatRange

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Utility

## Definition
```java
// Signature
public class FloatRange {
    // Static inner class
    public static class Constant implements IFloatRange { ... }

    // Static inner class
    public static class Normal implements IFloatRange { ... }
}
```

## Architecture & Concepts

The **FloatRange** class is a foundational utility within the procedural generation library. It serves as a namespace for concrete implementations of the **IFloatRange** strategy, providing a standardized way to define a numeric range from which a value can be sampled. Its primary purpose is to decouple the *definition* of a range (e.g., "between 10 and 50") from the *source of randomness or noise* used to select a value within that range.

This component embodies the Strategy Pattern. A system that requires a procedurally generated value does not need to know the specifics of the range; it only needs a contract (**IFloatRange**) that it can query with its own source of randomness, such as a world seed, a coordinate-based noise function, or a simple random number generator.

The two primary implementations provided are:
-   **FloatRange.Constant:** Represents a degenerate range with a single, fixed value. This is used when a value must conform to the **IFloatRange** interface but is not actually variable.
-   **FloatRange.Normal:** Represents a standard linear range between a minimum and maximum value. It maps an input value, typically from 0.0 to 1.0, to its configured output range.

## Lifecycle & Ownership
-   **Creation:** Instances of **FloatRange.Constant** or **FloatRange.Normal** are created on-demand by higher-level systems, such as biome descriptors, feature placement configurations, or entity attribute initializers. They are typically instantiated during the loading or configuration phase of a procedural generator.
-   **Scope:** These are lightweight, transient value objects. Their lifetime is tied to their owning configuration object. They do not persist beyond the generation task for which they were created.
-   **Destruction:** Objects are managed by the Java garbage collector. As they hold no external resources and are not registered with any central service, no explicit cleanup is required.

## Internal State & Concurrency
-   **State:** Instances are **immutable**. The internal fields (**result** for **Constant**; **min** and **range** for **Normal**) are set only during construction and cannot be modified thereafter. This design choice makes them highly predictable and reusable.
-   **Thread Safety:** The class and its inner implementations are inherently thread-safe due to their immutable state. The **getValue** methods are pure functions with respect to the object's internal state. They can be safely shared and called from multiple threads simultaneously without locks or synchronization, provided the supplied arguments (e.g., **Random**, **IDoubleCoordinateSupplier2d**) are themselves thread-safe.

## API Surface

The public contract is defined by the **IFloatRange** interface, which both **Constant** and **Normal** implement. The primary interaction point is the overloaded **getValue** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue(float v) | float | O(1) | Maps a normalized float (0.0 to 1.0) to the defined range. |
| getValue(Random random) | float | O(1) | Samples the range using the provided random number generator. |
| getValue(FloatSupplier supplier) | float | O(1) | Samples the range using a supplier for a normalized float. |
| getValue(seed, x, y, supplier) | float | O(1) | Samples the range using a 2D coordinate-based noise supplier. |
| getValue(seed, x, y, z, supplier) | float | O(1) | Samples the range using a 3D coordinate-based noise supplier. |

**WARNING:** The behavior of **getValue** is entirely dependent on the quality and distribution of the input source. The **FloatRange** object only performs a simple mapping; it does not generate randomness itself.

## Integration Patterns

### Standard Usage

**FloatRange** objects should be instantiated during configuration and passed to systems that perform procedural generation. The generating system then provides the source of randomness.

```java
// In a configuration or definition file
// Define a range for tree height, from 5.5 to 12.0 units.
IFloatRange treeHeightRange = new FloatRange.Normal(5.5f, 12.0f);

// In the world generation logic
// The generator uses its own seeded random source to get a value.
Random worldGenRandom = new Random(worldSeed);
float newTreeHeight = treeHeightRange.getValue(worldGenRandom);
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in Loops:** Do not create new **FloatRange** objects inside tight generation loops. They are immutable and can be safely reused. Instantiate them once and cache them.
    ```java
    // BAD: New object created for every single block
    for (int i = 0; i < 1000; i++) {
        IFloatRange badRange = new FloatRange.Normal(0f, 1f);
        float value = badRange.getValue(random);
    }

    // GOOD: Object is created once and reused
    IFloatRange goodRange = new FloatRange.Normal(0f, 1f);
    for (int i = 0; i < 1000; i++) {
        float value = goodRange.getValue(random);
    }
    ```
-   **Misusing Constant:** Do not use **FloatRange.Normal** to represent a fixed value. While `new FloatRange.Normal(5f, 5f)` works, it is less efficient and less clear than using `new FloatRange.Constant(5f)`.

## Data Pipeline

**FloatRange** acts as a transformation stage in a data pipeline, converting a raw noise or random value into a domain-specific value.

> Flow:
> Source of Randomness (e.g., **Random**, **IDoubleCoordinateSupplier3d**) -> **FloatRange.getValue()** -> Mapped Procedural Value (e.g., height, temperature, scale)

