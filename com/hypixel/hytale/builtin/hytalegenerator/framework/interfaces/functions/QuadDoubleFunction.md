---
description: Architectural reference for QuadDoubleFunction
---

# QuadDoubleFunction

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.interfaces.functions
**Type:** Utility

## Definition
```java
// Signature
@FunctionalInterface
public interface QuadDoubleFunction<R> {
```

## Architecture & Concepts
The QuadDoubleFunction interface defines a standardized contract for any function that accepts four double-precision floating-point numbers as input and produces a single object of a generic type R as output.

Within the Hytale Generator Framework, this interface is a fundamental building block for procedural generation algorithms, particularly those operating in three or four dimensions. It serves as a strategy pattern primitive, allowing complex generator systems to be composed of interchangeable mathematical or logical components. Common use cases include 4D noise sampling (x, y, z, w), 3D spatial queries with a fourth parameter for time or seed, or any computation requiring four independent floating-point inputs.

By abstracting the function signature, the framework can pass around behavior as data, enabling highly dynamic and configurable world generation pipelines.

### Lifecycle & Ownership
As a functional interface, QuadDoubleFunction itself has no lifecycle. The lifecycle pertains to the *implementing instance*, which is typically a lambda expression or a method reference.

- **Creation:** An instance is created implicitly whenever a lambda expression matching the interface's signature is defined and assigned to a variable or passed as a method argument. For example, `(x, y, z, w) -> x + y + z + w` creates a new instance.
- **Scope:** The lifetime of the instance is tied to the object that holds a reference to it. If passed to a long-lived generator service, it will persist with that service. If defined within a method scope for a temporary calculation, it becomes eligible for garbage collection once the method returns.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup requirements.

## Internal State & Concurrency
- **State:** The interface is inherently stateless. However, an implementing lambda can be stateful if it captures variables from its enclosing scope (i.e., it becomes a closure). State management and its consequences are the sole responsibility of the implementer.

- **Thread Safety:** The interface contract makes no guarantees about thread safety. Safety is entirely dependent on the implementation. If a lambda implementation modifies shared state (e.g., captured non-local variables, static fields), the calling code is responsible for ensuring proper synchronization. Most implementations, especially those performing pure mathematical calculations, are naturally thread-safe.

**WARNING:** Assume implementations are **not** thread-safe unless explicitly documented otherwise. Generator systems that execute these functions in parallel must ensure the implementations are pure or properly synchronized.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(double, double, double, double) | R | Varies | Executes the function's logic. Complexity is determined by the specific implementation. |

## Integration Patterns

### Standard Usage
This interface is intended to be used with lambda expressions to provide behavior to higher-order functions within the generator framework.

```java
// A hypothetical noise sampler that requires a 4D function
NoiseSampler4D sampler = new NoiseSampler4D();

// Provide a lambda implementation of QuadDoubleFunction to configure the sampler
// The lambda calculates a simple value based on four input coordinates
sampler.setNoiseFunction((x, y, z, w) -> {
    // Complex logic here...
    return x * Math.sin(y) + z * Math.cos(w);
});

// The sampler can now invoke the provided logic
Double result = sampler.getNoise(10.5, 20.1, 5.3, 0.75);
```

### Anti-Patterns (Do NOT do this)
- **Verbose Anonymous Classes:** Avoid implementing the interface with a full anonymous inner class when a concise lambda or method reference is sufficient. This adds unnecessary boilerplate.
- **Stateful Implementations in Parallel Systems:** Do not provide a stateful lambda (one that modifies external state) to a system that processes generation tasks in parallel without implementing explicit, correct synchronization. This is a primary source of race conditions and non-deterministic world generation.

## Data Pipeline
This interface acts as a processing node within a larger data pipeline, transforming coordinate or parameter data into a resulting value.

> Flow:
> Generator System -> Input Coordinates (double, double, double, double) -> **QuadDoubleFunction.apply()** -> Output Value (Type R) -> Voxel Data / Biome Map / etc.

