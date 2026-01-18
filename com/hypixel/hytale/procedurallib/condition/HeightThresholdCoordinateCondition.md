---
description: Architectural reference for HeightThresholdCoordinateCondition
---

# HeightThresholdCoordinateCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Transient / Strategy

## Definition
```java
// Signature
public class HeightThresholdCoordinateCondition implements ICoordinateCondition {
```

## Architecture & Concepts
The HeightThresholdCoordinateCondition is a predicate component within the procedural generation framework, designed to return true or false for a given 3D coordinate based on a height-sensitive threshold. It is a fundamental building block for creating vertically-aware features, such as biome transitions, ore veins, or cave systems that change with altitude.

Architecturally, this class implements the **Strategy Pattern**. It is a lightweight container that delegates the core logic of calculating a "threshold" value to an injected IHeightThresholdInterpreter dependency. This decouples the condition's evaluation framework from the specific algorithm used to determine the height threshold, allowing for immense flexibility. For example, different interpreters could define thresholds based on simple constants, noise functions, or complex biome-specific rules without requiring any change to this class.

The evaluation logic compares the threshold value returned by the interpreter against a deterministic, pseudo-random value generated from the coordinate and world seed. This technique ensures that for a given seed, the outcome is always the same, but introduces localized, organic-looking variations.

**WARNING:** The engine's coordinate system convention is critical here. The `eval(seed, x, y, z)` method passes its `y` argument as the `height` parameter to the interpreter. This implies that the **Y-axis is treated as the vertical axis** for all height-based calculations within this system.

## Lifecycle & Ownership
-   **Creation:** Instantiated directly by a higher-level procedural generation controller, such as a biome or feature definition. It is not managed by a central service registry or dependency injection container. Its creation is part of the *configuration* of a generation pass.
-   **Scope:** The object's lifetime is tied to its owning configuration. It is typically a short-lived, transient object that exists only for the duration of a specific world generation task.
-   **Destruction:** The object is eligible for garbage collection as soon as the procedural generation task completes and its parent configuration object goes out of scope. No manual cleanup is necessary.

## Internal State & Concurrency
-   **State:** **Immutable**. The class holds a single `final` reference to an IHeightThresholdInterpreter, which is set at construction. It contains no other mutable state.
-   **Thread Safety:** **Thread-safe**. Due to its immutable design, a single instance can be safely evaluated by multiple world generation threads concurrently without external locking. This is a critical feature for enabling high-performance, parallelized world generation.

    **WARNING:** The thread safety guarantee is contingent on the injected IHeightThresholdInterpreter also being thread-safe. Injecting a stateful or non-thread-safe interpreter will violate the concurrency contract of this class.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(seed, x, y) | boolean | O(1) | **Unsupported.** Throws UnsupportedOperationException. This condition is strictly 3D. |
| eval(seed, x, y, z) | boolean | O(N) | Evaluates the condition at a 3D coordinate. Complexity is dependent on the injected interpreter, but typically O(1) for noise-based lookups. |

## Integration Patterns

### Standard Usage
The class is intended to be constructed with a concrete interpreter implementation and used as a predicate within a generation loop.

```java
// 1. Obtain or create a concrete interpreter strategy
IHeightThresholdInterpreter noiseInterpreter = new NoiseBasedInterpreter(...);

// 2. Configure the condition with the chosen strategy
ICoordinateCondition condition = new HeightThresholdCoordinateCondition(noiseInterpreter);

// 3. Use the condition within a procedural generation algorithm
//    (e.g., iterating through a chunk's block column)
for (int y = 0; y < WORLD_HEIGHT; y++) {
    if (condition.eval(worldSeed, blockX, y, blockZ)) {
        // The condition is met at this height.
        // Place a specific block, spawn an entity, etc.
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Calling the 2D `eval` method:** Never call `eval(seed, x, y)`. It is not implemented and will crash the generation thread. This component is fundamentally incompatible with 2D evaluation contexts.
-   **Injecting Stateful Interpreters:** Do not inject an IHeightThresholdInterpreter that maintains mutable state if the condition will be used across multiple threads. This breaks the immutability and thread-safety contract, leading to severe and difficult-to-debug race conditions during world generation.

## Data Pipeline
This component acts as a conditional gate in a data flow. It does not transform data, but rather produces a boolean result based on its inputs.

> Flow:
> World Generator provides (Seed, X, Y, Z) -> **HeightThresholdCoordinateCondition**.eval() -> Delegates to `IHeightThresholdInterpreter` with (Seed, X, Z, Height=Y) -> Interpreter returns a threshold value -> Condition compares threshold against a deterministic hash of (Seed, X, Y, Z) -> Returns final boolean result to World Generator.

