---
description: Architectural reference for NoiseHeightThresholdInterpreter
---

# NoiseHeightThresholdInterpreter

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Transient

## Definition
```java
// Signature
public class NoiseHeightThresholdInterpreter implements IHeightThresholdInterpreter {
```

## Architecture & Concepts
The NoiseHeightThresholdInterpreter is a fundamental component within the procedural world generation framework. It functions as a composite interpreter that selects and blends between multiple underlying IHeightThresholdInterpreter instances based on a noise value.

Its primary role is to enable complex, organic transitions between different terrain generation rules. For example, a world generation definition can use this class to smoothly interpolate between a "plains" interpreter and a "mountains" interpreter. The selection is not random; it is determined by sampling a configured NoiseProperty at a given world coordinate. This allows for large, contiguous regions of one type to blend naturally into another.

Architecturally, this class implements a form of the **Strategy** or **Composite** pattern. It encapsulates an array of strategies (the IHeightThresholdInterpreter values) and uses a set of keys, driven by a noise context, to determine which strategy to apply or how to interpolate between them. This decouples the high-level transition logic from the low-level threshold calculations, promoting modular and reusable world generation definitions.

## Lifecycle & Ownership
- **Creation:** Instances are created during the parsing and initialization of world generation assets. It is expected to be instantiated via its public constructor, typically by a factory or parser that reads a procedural generation definition file.
- **Scope:** The object's lifetime is bound to the world generation configuration it is a part of. It is not a global singleton and persists only as long as its parent configuration is held in memory for a generation task.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection once the world generation process completes and all references to its configuration are released.

## Internal State & Concurrency
- **State:** This class is **effectively immutable**. All internal fields are final and are assigned exclusively during construction. The constructor performs pre-calculation of optimization values like lowestNonOne and highestNonZero. The behavior of the object does not change after it has been created.

- **Thread Safety:** Instances of NoiseHeightThresholdInterpreter are **thread-safe**. Due to its immutable nature, a single instance can be safely shared and accessed by multiple world generation worker threads simultaneously without external locking. All its computational methods, such as getThreshold, are pure functions with no side effects.

## API Surface
The public contract is focused on retrieving a threshold value for a given point in the world.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getThreshold(seed, x, z, height, context) | float | O(N) | Core method. Returns a threshold by selecting and blending interpreters based on the provided context. N is the number of keys. |
| getContext(seed, x, y) | double | O(1) | Samples the configured noise function to produce the context value used by getThreshold. Complexity depends on the underlying noise implementation. |
| getLowestNonOne() | int | O(1) | Returns the pre-computed minimum height at which a threshold can be non-one across all child interpreters. Used for optimization. |
| getHighestNonZero() | int | O(1) | Returns the pre-computed maximum height at which a threshold can be non-zero across all child interpreters. Used for optimization. |

## Integration Patterns

### Standard Usage
This class is intended to be configured and used by the world generation engine. A developer would typically define its parameters in a data file and let the engine instantiate it.

```java
// Conceptual example of engine-level usage
// Assume 'plains' and 'mountains' are other IHeightThresholdInterpreter instances

NoiseProperty transitionNoise = ... // Configure a noise function
float[] keys = new float[] { 0.4f, 0.6f };
IHeightThresholdInterpreter[] values = new IHeightThresholdInterpreter[] { plains, mountains };

// The engine creates the interpreter from the configuration
IHeightThresholdInterpreter interpreter = new NoiseHeightThresholdInterpreter(transitionNoise, keys, values);

// During world generation, the engine queries for a threshold
float threshold = interpreter.getThreshold(worldSeed, 1024.5, 2048.1, 64);
```

### Anti-Patterns (Do NOT do this)
- **Unsorted Keys:** The `keys` array provided to the constructor **must be sorted in ascending order**. The internal algorithm is a linear search that depends on this ordering for correct interpolation. Unsorted keys will lead to unpredictable and incorrect blending behavior.
- **Mismatched Array Lengths:** The constructor will throw an `IllegalStateException` if the `keys` and `values` arrays are not of equal length. This is a fatal configuration error.
- **Inconsistent Child Interpreter Lengths:** All IHeightThresholdInterpreter instances within the `values` array must report the same `length`. The constructor validates this and will throw an `IllegalStateException` if a mismatch is found.

## Data Pipeline
The NoiseHeightThresholdInterpreter acts as a conditional processing node within the larger world generation data flow. It does not manage a pipeline itself but is a critical step within one.

> Flow:
> World Generator Request (seed, x, y, z) -> **NoiseHeightThresholdInterpreter.getContext** -> Sample NoiseProperty -> Noise Value (Context) -> **NoiseHeightThresholdInterpreter.getThreshold** -> Linear Search on Keys -> Select/Interpolate Child Interpreters -> Final Threshold (float) -> World Generator Response

