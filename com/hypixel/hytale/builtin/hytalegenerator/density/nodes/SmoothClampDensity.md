---
description: Architectural reference for SmoothClampDensity
---

# SmoothClampDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class SmoothClampDensity extends Density {
```

## Architecture & Concepts
The SmoothClampDensity class is a foundational component within the procedural world generation system, specifically operating as a node in a **Density Graph**. Its primary function is to act as a mathematical filter that constrains an incoming density value to a specified range, but does so using a non-linear, smooth transition near the boundaries.

This is distinct from a hard clamp which would produce sharp, artificial edges in generated terrain. Instead, SmoothClampDensity uses hermite interpolation (via the Calculator utility) to gently ease values toward the minimum and maximum limits. This makes it an essential tool for shaping noise functions and other procedural data to create natural-looking features like plateaus, riverbeds, or altitude limits.

It is a processing node in a directed acyclic graph (DAG) of Density objects. It receives a single input value from an upstream node, transforms it, and passes the result downstream. The `min`, `max`, and `range` parameters define the transformation's behavior, where `range` controls the "softness" of the clamp.

## Lifecycle & Ownership
- **Creation:** Instances are typically created by a higher-level graph builder or a configuration parser during the initialization of a world generator. They are not intended for direct instantiation during the main game loop. The constructor requires valid configuration parameters, otherwise it will throw an IllegalArgumentException.
- **Scope:** The lifetime of a SmoothClampDensity object is tied to the Density Graph it is a part of. These graphs are generally constructed once per generation task (e.g., for a world chunk) and are discarded after the task is complete.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and has no explicit `destroy` or `close` method. It becomes eligible for collection as soon as the parent Density Graph is no longer referenced.

## Internal State & Concurrency
- **State:** This object is stateful. It maintains its configuration parameters (`min`, `max`, `range`) as immutable final fields. However, its connection to the upstream graph, the `input` field, is mutable and can be re-wired after construction via the `setInputs` method.

- **Thread Safety:** This class is **not thread-safe**. The `input` field can be modified at any time. If one thread calls `setInputs` while another thread is executing the `process` method, the behavior is undefined and may result in inconsistent density calculations.

    **Warning:** All modifications to a Density Graph, including calls to `setInputs`, must be completed before the graph is used for processing across multiple threads. The graph should be treated as immutable during the `process` phase.

## API Surface
The public API is minimal, focusing on graph construction and processing.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(D) | Processes the input node and applies the smooth clamp. Complexity is dependent on the depth (D) of the subgraph rooted at the input node. |
| setInputs(Density[] inputs) | void | O(1) | Wires the upstream node. Expects an array with at least one element; only the first element is used. |

## Integration Patterns

### Standard Usage
SmoothClampDensity is used as a wrapper around another Density node to constrain its output. The standard pattern involves chaining nodes together to form a complex generation pipeline.

```java
// Assume 'perlinNoise' is a pre-existing Density node generating values from -1.0 to 1.0
// We want to create a plateau between 0.2 and 0.6, with a soft transition zone of 0.1
Density perlinNoise = new PerlinNoise(...);

// Wrap the noise function with the smooth clamp
// min = 0.2, max = 0.6, range = 0.1
Density plateau = new SmoothClampDensity(0.2, 0.6, 0.1, perlinNoise);

// When 'plateau.process(context)' is called, it will first call 'perlinNoise.process(context)'
// and then smoothly clamp the result.
double finalValue = plateau.process(context);
```

### Anti-Patterns (Do NOT do this)
- **Invalid Configuration:** Do not construct with a `range` less than or equal to zero, or with `max` less than `min`. This will throw an `IllegalArgumentException` and halt generation.
- **Concurrent Modification:** Do not call `setInputs` on a node while any thread might be calling `process` on the same graph. This is a severe race condition.
- **Null Input Processing:** While the `process` method is null-safe and returns 0.0 if the input is null, this is almost always a sign of a misconfigured graph. A properly constructed graph should not have null inputs at processing time.

## Data Pipeline
The flow of data through this component is linear and unidirectional. It acts as a single stage in a larger density calculation pipeline.

> Flow:
> Upstream Density Node -> `process()` call -> Raw double value -> **SmoothClampDensity** -> `Calculator.smoothMin()` -> `Calculator.smoothMax()` -> Final clamped double value -> Downstream Consumer

