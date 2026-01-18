---
description: Architectural reference for DoubleRange
---

# DoubleRange

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Utility

## Definition
```java
// Signature
public class DoubleRange {
    // Contains static inner classes: Constant, Multiple, Normal
}
```

## Architecture & Concepts

The DoubleRange class serves as a namespace for a family of strategies that map a normalized input value (typically 0.0 to 1.0) to a target range. It is a fundamental component within the procedural generation library, designed to decouple the source of a value, such as a noise function or random number generator, from its final application in the game world.

This system employs the Strategy pattern, where each inner class (**Constant**, **Normal**, **Multiple**) implements a common contract, likely the IDoubleRange interface. This allows procedural systems, like biome or feature generators, to be configured with different mapping functions without altering their core logic. For example, the same noise supplier can be used to generate terrain height via a DoubleRange.Normal instance and to select a block type via a DoubleRange.Multiple instance.

The primary role of this component is to translate abstract procedural outputs into concrete game parameters like elevation, temperature, or resource density.

### Lifecycle & Ownership

-   **Creation:** Instances of the inner classes (Constant, Normal, Multiple) are value objects. They are typically instantiated directly during the setup of a procedural generation algorithm, often based on deserialized configuration data that defines world generation rules.
-   **Scope:** These objects are transient and lightweight. Their lifetime is tied to the procedural generator that owns them. They persist as long as the rule set they define is active.
--   **Destruction:** Managed entirely by the Java garbage collector. No manual cleanup or resource release is required.

## Internal State & Concurrency

-   **State:** All DoubleRange strategy objects are **immutable**. Their internal state (e.g., min, max, thresholds, values) is established exclusively at construction time via final fields. They do not cache results or modify their state during operation.
-   **Thread Safety:** These objects are inherently **thread-safe**. Due to their immutable nature and the absence of side effects in their methods, a single instance can be safely shared and accessed by multiple world generation worker threads simultaneously without requiring any external locking or synchronization.

## API Surface

The public contract is defined by the `getValue` method, which is overloaded to accept various value suppliers. This documentation assumes the common interface is IDoubleRange.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue(double v) | double | O(1) to O(N) | Maps a raw double value. Complexity is O(1) for Normal and Constant, but O(N) for Multiple, where N is the number of thresholds. |
| getValue(DoubleSupplier) | double | O(1) to O(N) | Maps a value from a generic DoubleSupplier. |
| getValue(Random) | double | O(1) to O(N) | Maps a value from a Random instance, implicitly calling nextDouble. |
| getValue(seed, x, y, IDoubleCoordinateSupplier2d) | double | O(1) to O(N) | Maps a value from a 2D coordinate-based noise function or supplier. |
| getValue(seed, x, y, z, IDoubleCoordinateSupplier3d) | double | O(1) to O(N) | Maps a value from a 3D coordinate-based noise function or supplier. |

## Integration Patterns

### Standard Usage

A procedural generator is configured with an IDoubleRange implementation. During generation, it invokes `getValue` with its noise supplier to determine a world parameter at a specific coordinate.

```java
// A noise function is the source of the normalized value
IDoubleCoordinateSupplier2d noise = new PerlinNoise(...);

// A DoubleRange instance defines the mapping rule (e.g., terrain height)
IDoubleRange heightRange = new DoubleRange.Normal(64.0, 192.0);

// At generation time, the range is applied to the noise output
double worldX = 102.5;
double worldZ = -50.0;
int seed = 12345;

double terrainHeight = heightRange.getValue(seed, worldX, worldZ, noise);
// terrainHeight is now a value between 64.0 and 192.0
```

### Anti-Patterns (Do NOT do this)

-   **Misconfigured Thresholds:** When using DoubleRange.Multiple, the `thresholds` array must be sorted in ascending order. The logic does not validate this assumption. Providing an unsorted array will lead to unpredictable and incorrect interpolation results.
-   **Array Length Mismatch:** For DoubleRange.Multiple, the `values` array should have a specific length relative to the `thresholds` array. Failure to adhere to the expected relationship will cause incorrect calculations or potential exceptions.
-   **Re-implementing Logic:** Do not attempt to read the min/max or thresholds/values from an instance to perform the mapping calculation externally. Always call the `getValue` method to ensure the correct, encapsulated logic is executed.

## Data Pipeline

DoubleRange acts as a stateless transformation stage within a larger procedural generation data pipeline. It does not initiate data flow but is a critical link in the chain.

> Flow:
> Coordinate Input -> Noise Supplier -> **DoubleRange (Mapping)** -> Game Parameter (e.g., Height) -> Voxel Generation Logic

