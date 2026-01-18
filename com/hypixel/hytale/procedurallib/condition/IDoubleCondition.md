---
description: Architectural reference for IDoubleCondition
---

# IDoubleCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Functional Interface / Contract

## Definition
```java
// Signature
public interface IDoubleCondition {
```

## Architecture & Concepts
The IDoubleCondition interface is a fundamental contract within the procedural generation library. It defines a stateless, predicate-based evaluation mechanism against a double-precision floating-point number. Its primary architectural role is to decouple procedural generation logic (e.g., "should a tree be placed here?") from the specific conditions that govern that logic (e.g., "is the noise value greater than 0.7?").

This interface is a key enabler for creating composable and configurable procedural systems. Algorithms can be written to operate against the IDoubleCondition contract, while the specific implementations (e.g., checking a range, a threshold, or a complex mathematical function) can be injected at runtime.

The default method `eval(int, IntToDoubleFunction)` is a critical integration point. It directly links the condition to seeded value generation, which is the cornerstone of deterministic and repeatable world generation. This allows a single integer seed to drive both the generation of a value (via the `IntToDoubleFunction`) and the evaluation of that value in a single, atomic operation.

## Lifecycle & Ownership
As an interface, IDoubleCondition does not have a concrete lifecycle. The lifecycle described here pertains to its *implementations*.

-   **Creation:** Implementations are typically instantiated as lambdas or anonymous inner classes during the setup and configuration of a procedural generator. They are often part of a larger configuration object that defines the rules for a specific biome or feature.
-   **Scope:** The lifetime of an IDoubleCondition implementation is tied to its owning configuration or procedural algorithm. In most scenarios, these are stateless and could be considered singletons within the scope of a generator's definition, but they are typically re-instantiated with each generator build.
-   **Destruction:** Implementations are eligible for garbage collection when the procedural generator or rule set that references them is no longer in scope.

## Internal State & Concurrency
-   **State:** The contract implicitly requires implementations to be **stateless and immutable**. The evaluation of a condition must produce the same result for the same input every time. Introducing state would violate the principle of deterministic procedural generation and is a severe anti-pattern.
-   **Thread Safety:** Implementations are expected to be inherently thread-safe due to their stateless nature. The `eval` methods should be pure functions, free of side effects. This is a critical property that allows the engine to parallelize chunks of procedural generation work without requiring locks or synchronization around condition evaluation.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(double var1) | boolean | O(1) | The core evaluation function. Returns true if the input value satisfies the condition. |
| eval(int seed, IntToDoubleFunction seedFunction) | boolean | O(1) + f(n) | A default convenience method. Generates a double from the seed using the provided function and evaluates it. Complexity depends on the supplied `seedFunction`. |

## Integration Patterns

### Standard Usage
The primary use case is to define a rule within a procedural system, often using a lambda for conciseness. The seeded `eval` is the preferred method for deterministic generation.

```java
// Example: A condition that is true 50% of the time, based on a seed.
// The IntToDoubleFunction would typically be a seeded random number generator or noise function.
IDoubleCondition highValueCondition = (val) -> val > 0.8;

IntToDoubleFunction noiseGenerator = (seed) -> Perlin.noise(seed);

// In a procedural algorithm, using a specific seed for a block position
boolean shouldPlaceFeature = highValueCondition.eval(blockPositionSeed, noiseGenerator);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** An implementation must not store state or change its behavior based on previous calls. This will break determinism and cause unpredictable generation results, which are extremely difficult to debug.
    ```java
    // DO NOT DO THIS
    class BadCondition implements IDoubleCondition {
        private int callCount = 0;
        public boolean eval(double val) {
            // Behavior changes on subsequent calls. This is forbidden.
            return val > 0.5 && (callCount++ % 2 == 0);
        }
    }
    ```
-   **Non-Deterministic Inputs:** Feeding values from non-deterministic sources (e.g., `System.currentTimeMillis()` or `new Random()`) into the `eval(double)` method for world generation will result in non-repeatable worlds. All inputs must be derived from the world seed.

## Data Pipeline
The interface acts as a processing node in a data flow, transforming a numeric value into a binary decision.

> Flow:
> World Seed -> Positional Seed -> **IntToDoubleFunction** (Noise) -> double value -> **IDoubleCondition.eval()** -> boolean result -> Generator Logic (e.g., Place Block)

