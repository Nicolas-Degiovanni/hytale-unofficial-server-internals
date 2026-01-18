---
description: Architectural reference for BasicHeightThresholdInterpreter
---

# BasicHeightThresholdInterpreter

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Transient

## Definition
```java
// Signature
public class BasicHeightThresholdInterpreter implements IHeightThresholdInterpreter {
```

## Architecture & Concepts
The BasicHeightThresholdInterpreter is a foundational component within the procedural world generation system. Its primary function is to translate a sparse set of height-based control points into a dense, continuous lookup table. This allows world generation designers to define simple rules, such as "at height 64, the threshold is 0.5; at height 128, it is 0.9," and the interpreter will calculate the appropriate threshold for every integer height between those points using linear interpolation.

This class embodies a classic space-for-time tradeoff. It performs a one-time, potentially expensive calculation in its constructor to pre-compute and cache an entire array of threshold values. Subsequent queries via the getThreshold method are then reduced to trivial, O(1) array lookups. This design is critical for performance in the world generation pipeline, where this calculation might be needed millions of time per chunk.

It serves as a data structure that represents a vertical gradient, which is then used by other procedural conditions to determine the placement of features, biomes, or materials based on altitude.

### Lifecycle & Ownership
-   **Creation:** Instantiated by higher-level procedural generation services, typically when parsing world generation configuration files. The constructor arguments, positions and thresholds, are sourced directly from these asset definitions.
-   **Scope:** The object's lifetime is bound to the procedural rule or condition that created it. It is not a global singleton. It persists as long as its parent configuration is loaded, often for the duration of a world generation session for a specific dimension or biome set.
-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It is eligible for collection once the parent procedural rule is unloaded.

## Internal State & Concurrency
-   **State:** The internal state is **effectively immutable** after the constructor completes. The core data structure, the interpolatedThresholds array, and the cached boundary indices (lowestNonOne, highestNonZero) are populated once and are never mutated again.
-   **Thread Safety:** This class is **thread-safe**. Due to its immutable nature post-construction, a single instance can be safely shared and accessed by multiple world generation worker threads simultaneously without any external locking or synchronization. This is a crucial feature for a parallelized world generation engine.

## API Surface
The public contract is designed for high-performance lookups.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BasicHeightThresholdInterpreter(positions, thresholds, length) | constructor | O(N*M) | Creates and pre-computes the lookup table. N is length, M is positions.length. Throws IllegalArgumentException if array lengths mismatch. |
| getThreshold(seed, x, y, height, context) | float | O(1) | Returns the pre-calculated interpolated threshold for a given height. This is the primary query method. |
| getLowestNonOne() | int | O(1) | Returns the lowest pre-calculated height at which the threshold is less than 1.0. |
| getHighestNonZero() | int | O(1) | Returns the highest pre-calculated height at which the threshold is greater than 0.0. |
| getLength() | int | O(1) | Returns the total vertical size of the lookup table. |

**Warning:** The getContext method is a stub in this implementation, always returning 0.0. It exists to satisfy the IHeightThresholdInterpreter interface and is intended for extension by more complex interpreters that might factor in horizontal noise.

## Integration Patterns

### Standard Usage
This class should be instantiated once per procedural rule and reused for all subsequent checks related to that rule. It is a data object, not a service.

```java
// Typically loaded from a world generation configuration file
int[] yPositions = new int[]{64, 128, 192};
float[] thresholds = new float[]{0.0f, 0.8f, 0.2f};
int worldHeight = 256;

// Create the interpreter once
IHeightThresholdInterpreter interpreter = new BasicHeightThresholdInterpreter(yPositions, thresholds, worldHeight);

// In the world generation loop for a vertical column...
for (int y = 0; y < worldHeight; y++) {
    float threshold = interpreter.getThreshold(seed, x, z, y);
    float noiseValue = Perlin.getNoise(x, y, z);

    if (noiseValue > threshold) {
        // Place a specific block or feature
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Per-Block Instantiation:** Never create a new BasicHeightThresholdInterpreter inside a tight loop (e.g., for every block). The constructor performs a full interpolation pass and is too expensive for frequent invocation.
-   **Ignoring Cached Boundaries:** Do not manually iterate to find the active range of the interpreter. Use getLowestNonOne and getHighestNonZero to quickly cull entire sections of a world column from processing.

## Data Pipeline
The interpreter acts as a transformation step, converting declarative configuration data into a high-performance procedural generation component.

> Flow:
> World Gen Asset (JSON/HOCON) -> Config Deserializer -> **BasicHeightThresholdInterpreter (Constructor)** -> Cached instance held by a Biome or Feature Rule -> World Generator queries **getThreshold(y)** -> Threshold value is used in a comparison -> Block Placement Decision

