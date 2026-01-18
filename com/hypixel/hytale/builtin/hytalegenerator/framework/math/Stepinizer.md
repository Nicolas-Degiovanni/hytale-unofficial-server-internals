---
description: Architectural reference for Stepinizer
---

# Stepinizer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Transient

## Definition
```java
// Signature
public class Stepinizer implements Function<Double, Double>, Double2DoubleFunction {
```

## Architecture & Concepts
The Stepinizer is a stateful, configurable mathematical function object, or functor. Its primary role within the world generation framework is to transform a continuous input signal, such as a value from a noise generator, into a discrete, stepped output. This process is also known as quantization.

This component is fundamental for creating terraced or layered geological features. By quantizing the smooth gradients of noise fields, developers can produce distinct plateaus, cliffs, and strata in the terrain. The class provides fine-grained control over the size of each step, the steepness of the transition between steps, and the degree of smoothing at the edges, allowing for both hard, blocky transitions and softer, more organic ones.

It acts as a processing node in a larger procedural generation pipeline, typically consuming data from noise sources and feeding its output to systems responsible for material or height map assignment.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor: `new Stepinizer()`. It is not managed by a service locator or dependency injection container. Ownership is assumed by the creating class, typically a higher-level world generation algorithm.
- **Scope:** The object's lifetime is transient and typically confined to the scope of the method or generation pass in which it was created. It is designed to be configured once and used for a series of related calculations.
- **Destruction:** The object is managed by the Java Garbage Collector. It holds no native resources and requires no explicit cleanup. It is eligible for collection as soon as it falls out of scope.

## Internal State & Concurrency
- **State:** The Stepinizer is a **highly mutable** object. Its behavior is entirely defined by its internal fields: stepSize, slope, topSmooth, and bottomSmooth. These values are configured post-construction via a fluent API.

- **Thread Safety:** This class is **not thread-safe**. Its internal state is unprotected, and there are no synchronization mechanisms. Concurrent calls to its configuration methods (e.g., setStep) and its application method (get) will lead to race conditions and unpredictable results.

    **WARNING:** A single Stepinizer instance must not be shared across threads without external synchronization. For parallel generation tasks, each thread or worker must create and configure its own private instance.

## API Surface
The public API is designed as a fluent builder for configuration, followed by application of the function.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setSmooth(top, bottom) | Stepinizer | O(1) | Configures the smoothing factor for the top and bottom edges of a step. Throws IllegalArgumentException for non-positive values. |
| setEdgeSlope(slope) | Stepinizer | O(1) | Sets the steepness of the transition between steps. Throws IllegalArgumentException for negative values. |
| setStep(size) | Stepinizer | O(1) | Defines the height or size of each discrete step. Throws IllegalArgumentException for negative values. |
| get(x) | double | O(1) | Executes the step function on the input value. This is the primary calculation method. |
| apply(x) | double | O(1) | An alias for get(x), satisfying the Java Function interface contract. |

## Integration Patterns

### Standard Usage
The intended pattern is to instantiate, configure via the fluent setters, and then use the object to process a stream of data within a single-threaded context.

```java
// 1. Instantiate and configure the function object
Stepinizer terrainTerracer = new Stepinizer()
    .setStep(16.0)
    .setEdgeSlope(2.5)
    .setSmooth(0.8, 0.8);

// 2. Apply the function to input data, e.g., from a noise generator
for (int x = 0; x < width; x++) {
    double rawNoise = noiseGenerator.getNoise(x);
    double terracedValue = terrainTerracer.get(rawNoise);
    // ... use terracedValue for terrain height
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not modify a Stepinizer's state from one thread while another thread is calling its get or apply method. This will result in calculation errors.
- **State Re-use without Reset:** Do not reuse a single Stepinizer instance for different generation features without re-configuring it. Its state persists between calls to get. This can lead to one feature's configuration unintentionally affecting another's.
- **Passing to Unsafe Consumers:** Do not pass a configured Stepinizer to a system that may later modify its state. Once configured for a specific task, it should be treated as immutable.

## Data Pipeline
The Stepinizer is a transformation stage in a data processing pipeline. It does not produce or consume data on its own.

> Flow:
> Continuous Signal (e.g., Noise Value) -> **Stepinizer.get(value)** -> Quantized Signal (e.g., Terrain Height)

