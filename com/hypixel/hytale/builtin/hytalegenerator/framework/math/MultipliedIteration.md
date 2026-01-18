---
description: Architectural reference for MultipliedIteration
---

# MultipliedIteration

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Utility

## Definition
```java
// Signature
public class MultipliedIteration {
```

## Architecture & Concepts
The MultipliedIteration class is a stateless, mathematical utility designed to solve problems related to geometric progression and iterative decay. It is a foundational component primarily intended for use within procedural content generation (PCG) systems, such as the Hytale world generator.

Its core function is to determine a multiplicative factor that, when applied repeatedly, will reduce a starting value to a target end value over a specified number of iterations. This is a common requirement in algorithms that need to smoothly transition a parameter over a fixed number of steps. For example, it can be used to calculate the rate at which a terrain brush's influence should diminish or the probability of a biome feature spawning as distance from an origin point increases.

This class encapsulates a pure, deterministic calculation and has no dependencies on other engine systems. It acts as a low-level building block for more complex generation logic.

### Lifecycle & Ownership
- **Creation:** As a static utility class, MultipliedIteration is never instantiated. Its bytecode is loaded by the JVM ClassLoader when first referenced by another component.
- **Scope:** Application-wide. The static methods are available globally for the entire duration of the application's runtime once the class is loaded.
- **Destruction:** The class is unloaded by the JVM during the final stages of application shutdown. There is no manual memory management required.

## Internal State & Concurrency
- **State:** This class is completely stateless. It contains no member fields and all calculations are performed exclusively on the arguments provided to its static methods. It is therefore inherently immutable.
- **Thread Safety:** MultipliedIteration is unconditionally thread-safe. Due to its stateless nature, there are no shared resources, locks, or potential for race conditions. Its methods can be invoked concurrently from any number of threads without synchronization or side effects.

## API Surface
The public API consists of two static methods for performing the core calculations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateMultiplier(start, end, iterations, precision) | double | O(1/precision * log(start/end)) | Calculates the multiplicative factor required to decay from a start to an end value. This is the primary method. Throws IllegalArgumentException if parameters are invalid. |
| calculateIterations(multiplier, start, end) | int | O(log(start/end)) | A helper method that calculates the number of iterations required to decay from a start to an end value given a fixed multiplier. |

**WARNING:** The `calculateMultiplier` method employs an iterative search. Its performance is directly proportional to the inverse of the supplied `precision`. A very small precision value can result in a significant number of internal loop cycles and impact performance.

## Integration Patterns

### Standard Usage
The intended pattern is to calculate a multiplier once at the beginning of a generation process and reuse the result. The calling system is responsible for caching and reusing the calculated multiplier.

```java
// Example: Calculating a decay factor for a procedural process
double startInfluence = 100.0;
double endInfluence = 5.0;
int steps = 10;
double searchPrecision = 0.001;

// Calculate the multiplier once and store it
double decayMultiplier = MultipliedIteration.calculateMultiplier(
    startInfluence,
    endInfluence,
    steps,
    searchPrecision
);

// Use the cached multiplier repeatedly in a generation loop
double currentInfluence = startInfluence;
for (int i = 0; i < steps; i++) {
    // Apply procedural logic with currentInfluence
    System.out.println("Step " + i + ": Influence = " + currentInfluence);

    // Apply the decay for the next step
    currentInfluence *= decayMultiplier;
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new MultipliedIteration()`. This is a utility class with only static methods and should never be instantiated.
- **Recalculation in Loops:** Avoid calling `calculateMultiplier` inside a high-frequency loop with the same parameters. This is computationally wasteful. Calculate the value once and cache it.
- **Excessive Precision:** Do not provide an unnecessarily small value for the `precision` parameter in `calculateMultiplier`. This can cause performance degradation with no meaningful improvement in the result for most use cases.

## Data Pipeline
MultipliedIteration is not part of a data streaming pipeline but rather acts as a pure function within a larger algorithmic process. It takes numerical inputs and produces a numerical output.

> **Flow within a World Generator:**
>
> Generator Configuration -> **MultipliedIteration.calculateMultiplier** -> Cached Multiplier -> Terrain Generation Loop -> Final World Data

