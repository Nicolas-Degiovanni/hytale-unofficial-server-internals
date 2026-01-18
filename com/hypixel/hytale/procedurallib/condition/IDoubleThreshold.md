---
description: Architectural reference for IDoubleThreshold
---

# IDoubleThreshold

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IDoubleThreshold {
```

## Architecture & Concepts
The IDoubleThreshold interface defines a fundamental contract within the procedural generation library. It represents a simple, stateless predicate used to evaluate a condition against one or two double-precision floating-point numbers.

Architecturally, this interface serves as a strategy pattern component. It decouples the high-level procedural algorithms (e.g., biome placement, terrain feature generation) from the specific conditional logic used to make decisions. By passing implementations of IDoubleThreshold to these algorithms, developers can inject custom evaluation logic without modifying the core generation code. This promotes reusability and simplifies the creation of complex, layered procedural rules.

Its primary application is in evaluating outputs from noise functions or other data sources to determine if they meet a specific criterion, such as falling within a valid range for temperature, humidity, or elevation.

## Lifecycle & Ownership
As an interface, IDoubleThreshold itself does not have a lifecycle. The lifecycle and ownership pertain to its concrete implementations.

- **Creation:** Implementations are typically instantiated as needed by the procedural generation system. They are often created as anonymous inner classes, lambdas, or stateless singleton utility objects. They are not managed by a central registry or dependency injection framework.
- **Scope:** The scope of an implementation is almost always transient and local to the specific generation task that requires it. An instance typically lives only for the duration of a single method call or algorithm execution.
- **Destruction:** Implementations are garbage collected once the procedural task that created them completes and no longer holds a reference. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** The contract strongly implies that all implementations should be **stateless**. The `eval` methods operate solely on the arguments provided. Storing mutable state within an implementation is a severe anti-pattern that can lead to non-deterministic and incorrect generation results.
- **Thread Safety:** Because implementations are expected to be stateless, they are inherently **thread-safe**. The procedural generation engine may execute these evaluations concurrently across multiple threads without risk of race conditions, provided the contract of statelessness is upheld.

## API Surface
The public contract consists of two evaluation methods, allowing for both single-value and two-value comparisons.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(double var1) | boolean | O(1) | Evaluates a condition against a single input value. |
| eval(double var1, double var3) | boolean | O(1) | Evaluates a condition against two input values, often representing a value and a context (e.g., value and its maximum). |

## Integration Patterns

### Standard Usage
Implementations of IDoubleThreshold are passed to procedural algorithms to control their decision-making process. The caller provides the logic, and the algorithm executes it at the appropriate point in the generation pipeline.

```java
// Example: A procedural function that uses the interface
public void generateFeatures(NoiseMap heightMap, IDoubleThreshold placementCondition) {
    for (int x = 0; x < 128; x++) {
        double heightValue = heightMap.getValue(x);
        // The algorithm invokes the provided condition
        if (placementCondition.eval(heightValue)) {
            // Place a feature
        }
    }
}

// Usage: Defining a condition with a lambda
IDoubleThreshold isAboveSeaLevel = (height) -> height > 0.5;
generator.generateFeatures(myHeightMap, isAboveSeaLevel);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Never create an implementation that stores mutable state. This breaks thread safety and can cause unpredictable, non-reproducible world generation.
- **Heavy Computation:** The `eval` methods are expected to be extremely fast, typically simple comparisons. Do not perform file I/O, network requests, or complex calculations within an implementation. Such logic belongs higher up in the generation pipeline.

## Data Pipeline
IDoubleThreshold acts as a gate or filter within a larger data processing flow. It consumes a numerical value and produces a binary decision that dictates the subsequent path of the pipeline.

> Flow:
> Noise Function Output (double) -> **IDoubleThreshold.eval()** -> Boolean Decision -> Conditional Logic (e.g., Biome Selection, Block Placement)

