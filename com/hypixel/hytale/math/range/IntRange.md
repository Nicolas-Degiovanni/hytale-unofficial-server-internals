---
description: Architectural reference for IntRange
---

# IntRange

**Package:** com.hypixel.hytale.math.range
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class IntRange {
```

## Architecture & Concepts
The IntRange class is a fundamental mathematical primitive used throughout the engine to represent a closed, inclusive interval of integers, such as [5, 10]. It serves as a core data structure for any system requiring bounded numerical values, including procedural generation, configuration parsing, and AI behavior parameters.

Its primary architectural role is to encapsulate the concept of a minimum and maximum boundary, providing a standardized way to query values within that range. The class includes a pre-calculated `range` field, which is a performance optimization to avoid repeated subtraction operations during frequent calls to the `getInt` method.

A critical integration point is the static `CODEC` field. This exposes an IntRangeArrayCodec, signaling that IntRange objects are designed to be serialized and deserialized as part of the engine's broader data management and configuration loading systems.

## Lifecycle & Ownership
- **Creation:** IntRange instances are created on-demand. They are typically instantiated by higher-level systems that parse configuration files (using the associated `CODEC`) or by game logic that requires a temporary numerical boundary for an algorithm.
- **Scope:** The object's lifetime is strictly tied to its owner. If it is a field within a larger configuration object, it persists as long as that configuration is loaded. If created locally within a method, it is eligible for garbage collection once the method scope is exited. It is a lightweight, transient object.
- **Destruction:** Management is handled entirely by the Java Garbage Collector. No manual resource cleanup is necessary.

## Internal State & Concurrency
- **State:** The state of an IntRange object is **Mutable**. The `setInclusiveMin` and `setInclusiveMax` methods directly modify the internal boundaries and re-calculate the cached `range` value.

- **Thread Safety:** This class is **NOT thread-safe**. Due to its mutable nature, concurrent access can lead to severe data corruption and unpredictable behavior.

    **WARNING:** If one thread calls a setter (e.g., `setInclusiveMax`) while another thread calls `getInt`, a race condition can occur. The `getInt` method may use a stale `range` value with an updated `inclusiveMin` or `inclusiveMax`, resulting in incorrect calculations or out-of-bounds values. Any shared instance of IntRange **must** be protected by external synchronization mechanisms like locks or be confined to a single thread.

## API Surface
The public API provides methods for defining the range and mapping a normalized floating-point value to a discrete integer within it.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| IntRange(int, int) | constructor | O(1) | Constructs a range with the specified inclusive minimum and maximum. |
| setInclusiveMin(int) | void | O(1) | Updates the lower boundary and recalculates the internal range cache. |
| setInclusiveMax(int) | void | O(1) | Updates the upper boundary and recalculates the internal range cache. |
| getInt(float factor) | int | O(1) | Maps a normalized factor (0.0 to 1.0) to an integer within the range. |
| includes(int value) | boolean | O(1) | Performs a fast check to determine if a value falls within the boundaries. |

## Integration Patterns

### Standard Usage
IntRange is most commonly used to define a parameter from a configuration file and then sample values from it during gameplay.

```java
// Assume 'configRange' is deserialized from a data file
IntRange monsterCountRange = config.getMonsterCountRange(); // e.g., [5, 10]

// Use a random factor to determine the number of monsters to spawn
float randomFactor = Random.nextFloat(); // e.g., 0.75
int monstersToSpawn = monsterCountRange.getInt(randomFactor);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never share and modify an IntRange instance across multiple threads without explicit locking. This is the most critical anti-pattern and will lead to difficult-to-debug race conditions.

- **Invalid Range Definition:** The class does not internally validate that `inclusiveMin <= inclusiveMax`. Creating a range where the minimum is greater than the maximum will result in a negative cached `range` value, causing `getInt` to produce unexpected and incorrect results. Always ensure valid boundaries upon creation.

## Data Pipeline
IntRange is typically a consumer of data from configuration systems and a provider of data for game logic systems.

> Flow:
> Configuration File (JSON, etc.) -> IntRangeArrayCodec -> **IntRange Instance** -> Procedural Content Generator / AI System -> Game State

