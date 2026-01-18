---
description: Architectural reference for PointConsumer
---

# PointConsumer<T>

**Package:** com.hypixel.hytale.procedurallib.logic.point
**Type:** Functional Interface / Contract

## Definition
```java
// Signature
public interface PointConsumer<T> {
```

## Architecture & Concepts
The PointConsumer interface defines a fundamental contract within the procedural generation library. It embodies the "Visitor" or "Callback" pattern, providing a mechanism for processing a stream of generated points without first collecting them into a memory-intensive collection.

This abstraction is critical for performance and memory efficiency. Procedural generators, such as noise evaluators or shape rasterizers, can produce millions of points. Instead of returning a massive list, these systems "push" each generated point directly to a PointConsumer implementation. This decouples the point *generation* logic from the point *consumption* logic, allowing for flexible and efficient data pipelines. For example, one consumer might place blocks in the world, while another might simply collect points into a debug visualization.

The generic parameter T allows the consumer to receive not just spatial coordinates (var1, var3) but also an associated data value, such as density, biome type, or a material identifier.

## Lifecycle & Ownership
As an interface, PointConsumer itself has no lifecycle. The lifecycle and ownership semantics apply to the **implementing classes**.

- **Creation:** Instances are created by the system that needs to process generated points. They are typically instantiated as anonymous inner classes or lambdas immediately before a procedural generation task is invoked.
- **Scope:** The scope of a PointConsumer is almost always transient and task-specific. It lives only for the duration of the call to the generator function it is passed to. It is a short-lived object designed to handle a single, discrete generation operation.
- **Destruction:** The implementing object is eligible for garbage collection as soon as the generator function completes and releases its reference to the consumer.

## Internal State & Concurrency
- **State:** The interface is stateless. However, implementations are almost always **stateful**. A common pattern is for a PointConsumer implementation to accumulate results into an internal collection or modify some external state (e.g., a world chunk).
- **Thread Safety:** This interface is **not inherently thread-safe**. The responsibility for ensuring thread safety lies entirely with the implementer. Procedural generators may be parallelized and can invoke the `accept` method concurrently from multiple worker threads.

**WARNING:** Implementations that modify shared state (e.g., adding to a `java.util.ArrayList`) **must** use appropriate synchronization mechanisms (e.g., locks, concurrent collections) to prevent data corruption and race conditions. Failure to do so will lead to unpredictable and difficult-to-diagnose bugs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(double x, double y, T value) | void | O(N) | Called by a generator for each point. The complexity is determined by the implementation (N). |

## Integration Patterns

### Standard Usage
The primary pattern is to provide a lambda or anonymous class implementation to a function that generates points. This allows for direct, in-place processing of the data stream.

```java
// A generator function (hypothetical) might accept this consumer
PointGenerator generator = getPointGenerator();
List<PointData> collectedPoints = new ArrayList<>();

// The consumer is implemented as a lambda to collect points into a list
generator.generatePoints(
    (x, y, value) -> {
        collectedPoints.add(new PointData(x, y, value));
    }
);
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** The `accept` method is often called within a tight, performance-critical loop. Never perform blocking I/O, complex calculations, or any long-running operations within this method. Doing so will severely degrade or completely stall the procedural generation pipeline.
- **Unsynchronized State Modification:** As noted under Concurrency, modifying a non-thread-safe collection or shared object from within `accept` without proper locking is a severe anti-pattern that will cause data corruption when used with a parallel generator.

## Data Pipeline
The PointConsumer is a terminal stage or a transformation step in a procedural generation data flow.

> Flow:
> Procedural Generator (e.g., NoiseField, ShapeRasterizer) -> **PointConsumer.accept(x, y, value)** -> World State Update / Data Collection / Further Processing

