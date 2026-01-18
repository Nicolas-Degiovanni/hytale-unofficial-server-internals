---
description: Architectural reference for BiDouble2DoubleFunction
---

# BiDouble2DoubleFunction

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.interfaces.functions
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface BiDouble2DoubleFunction {
   double apply(double var1, double var3);
}
```

## Architecture & Concepts
BiDouble2DoubleFunction is a specialized, performance-oriented functional interface. It serves as a formal contract for any function that accepts two primitive double-precision floating-point numbers as input and produces a single primitive double as output.

Architecturally, this interface is a cornerstone of the functional programming paradigm adopted within the Hytale world generation framework. It allows complex algorithms, such as noise composition, terrain blending, and biome attribute calculation, to be parameterized by behavior. Instead of hard-coding mathematical formulas, core generator services accept implementations of BiDouble2DoubleFunction, effectively decoupling the algorithmic structure from the specific mathematical operations.

The explicit use of primitive `double` types is a critical performance optimization. It entirely avoids the overhead associated with autoboxing and unboxing the `Double` wrapper class, which can have a significant performance impact in tight loops that execute millions or billions of times during world generation.

## Lifecycle & Ownership
As a functional interface, BiDouble2DoubleFunction itself does not have a lifecycle. However, instances that implement this interface (typically lambda expressions or method references) do.

- **Creation:** Instances are created ad-hoc at the call site where a function is required. They are not managed by a service locator or dependency injection framework. The Java compiler synthesizes an anonymous class to implement the lambda expression.
- **Scope:** The lifetime of an instance is governed by standard Java garbage collection rules. If a lambda is passed as a method argument, it typically becomes eligible for garbage collection after the method returns, unless it is captured by a longer-lived object (e.g., stored in a field).
- **Destruction:** There is no explicit destruction. The Java Garbage Collector reclaims the memory of an instance when it is no longer reachable.

## Internal State & Concurrency
- **State:** The interface contract implies statelessness. Implementations are expected to be pure functions, where the output depends solely on the input arguments. Lambdas that capture and modify external mutable state (i.e., are not pure) violate this expectation and are a significant source of architectural risk.
- **Thread Safety:** The interface itself is inherently thread-safe. The thread safety of a given *implementation* is entirely dependent on its definition. A stateless lambda, such as `(a, b) -> a * b`, is perfectly thread-safe. An implementation that captures and mutates a shared object is **not** thread-safe and will cause severe data corruption if used in the parallelized sections of the world generator.

**WARNING:** All implementations of this interface intended for use in the core generation pipeline **must** be stateless and thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(double, double) | double | Implementation-dependent | Executes the defined function. The complexity is determined by the lambda body, but is typically O(1) for simple arithmetic. |

## Integration Patterns

### Standard Usage
This interface is used to pass behavior into higher-order functions. A common use case is combining multiple noise layers or blending terrain features, where the specific blending mode is supplied as a lambda.

```java
// A utility method accepts a BiDouble2DoubleFunction to blend two noise values.
public double blendNoise(double noiseA, double noiseB, BiDouble2DoubleFunction blendMode) {
    return blendMode.apply(noiseA, noiseB);
}

// Usage:
// Define a simple additive blending function.
BiDouble2DoubleFunction additiveBlend = (a, b) -> a + b;

// Pass the behavior into the algorithm.
double result = blendNoise(0.75, 0.25, additiveBlend);

// Alternatively, use a lambda directly at the call site.
double multiplicativeResult = blendNoise(0.75, 0.25, (a, b) -> a * b);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Never define a lambda that modifies state outside of its own scope. This creates non-deterministic and unpredictable behavior, which is catastrophic for a procedural generator.
    ```java
    // DO NOT DO THIS
    List<Double> history = new ArrayList<>();
    BiDouble2DoubleFunction statefulFunc = (a, b) -> {
        history.add(a); // Modifies external state - NOT THREAD SAFE
        return a + b;
    };
    ```
- **Ignoring Performance:** While flexible, avoid creating overly complex or computationally expensive lambdas for this interface. These functions are often called in the tightest loops of the generator, and any inefficiency is magnified exponentially.

## Data Pipeline
BiDouble2DoubleFunction does not process a stream of data itself; rather, it acts as a pluggable computational step within a larger data generation pipeline.

> Flow:
> Generator Service (e.g., Noise Sampler) -> Provides two double values -> **BiDouble2DoubleFunction Implementation** -> Returns one double value -> Value used for further calculation (e.g., block type determination)

