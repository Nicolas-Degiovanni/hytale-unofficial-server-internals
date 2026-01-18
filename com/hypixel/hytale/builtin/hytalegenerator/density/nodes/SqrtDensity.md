---
description: Architectural reference for SqrtDensity
---

# SqrtDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class SqrtDensity extends Density {
```

## Architecture & Concepts
The SqrtDensity class is a specialized node within the procedural world generation's density function graph. It acts as a mathematical transformer, applying a signed square root function to the output of an upstream density node. Its primary role is to modify a density field in a non-linear way, which can be used to create sharper or smoother transitions in terrain features, cave formations, or biome distributions.

Architecturally, this class embodies the Composite and Decorator patterns. It is a leaf in a larger tree-like structure of Density objects, where each node processes a value and passes it on. The "signed" square root implementation is a notable feature; it handles negative inputs by calculating the square root of the absolute value and then reapplying the original sign. This ensures the function is defined across all real numbers and preserves the negative domain of the input density field, a critical behavior for certain generative algorithms.

This component is not a standalone service but a building block, intended to be composed with other Density nodes to create complex, emergent generative behavior.

### Lifecycle & Ownership
- **Creation:** SqrtDensity instances are not meant to be manually instantiated in game logic. They are created by a world generator's graph parser during the initialization phase, typically by deserializing a world generation configuration file.
- **Scope:** The lifetime of a SqrtDensity object is bound to the lifecycle of the parent density graph it belongs to. It persists as long as the world generator instance is active.
- **Destruction:** The object is managed by the Java garbage collector. It is de-referenced and cleaned up when the world generator configuration is unloaded or replaced. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The class holds a single mutable field, *input*, which is a reference to the upstream Density node. This reference can be changed after construction via the setInputs method, making the object's behavior mutable.

- **Thread Safety:** This class is **not** thread-safe for mutation but is safe for concurrent processing under specific conditions.
    - The process method is re-entrant and depends only on its arguments and the immutable state of its *input* node during the call. Multiple threads can safely invoke process concurrently on the same SqrtDensity instance, provided the density graph is not being modified.
    - **WARNING:** The setInputs method is a mutating operation and is not synchronized. Calling setInputs while the density graph is being processed by other threads will lead to race conditions and undefined behavior. All graph wiring must be completed in a single-threaded context before the world generator begins its work.

## API Surface
The public contract is minimal, designed for integration into the density graph framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SqrtDensity(Density input) | constructor | O(1) | Constructs the node, wiring its upstream input. |
| process(Density.Context context) | double | O(N) | Processes the input node and applies the signed square root. Complexity is dominated by the upstream input node's process call (O(N)). |
| setInputs(Density[] inputs) | void | O(1) | Re-wires the node's input. Expects an array with at least one element. **WARNING:** Not thread-safe. |

## Integration Patterns

### Standard Usage
SqrtDensity is used as a wrapper around another Density node to transform its output. It is almost always defined declaratively in a configuration and instantiated by a factory or parser.

```java
// Concept: Building a simple density graph programmatically.
// In practice, this graph is usually loaded from a config file.

// 1. Define a base noise function (the input)
Density sourceNoise = new SimplexNoiseDensity(/* parameters */);

// 2. Wrap it with SqrtDensity to modify its output curve
Density transformedNoise = new SqrtDensity(sourceNoise);

// 3. A world generator system processes a point in the world
//    using the final node of the graph.
double finalValue = transformedNoise.process(densityContext);
```

### Anti-Patterns (Do NOT do this)
- **Dynamic Re-Wiring:** Do not call setInputs on a SqrtDensity node after the world generator has started processing chunks. The density graph is expected to be immutable during generation.
- **Null Input Dependency:** The code gracefully handles a null *input* by returning 0.0. Do not rely on this as a logical switch. A null input almost always indicates a configuration error in the density graph that should be fixed at the source.
- **Direct Instantiation in Game Logic:** Avoid creating SqrtDensity instances directly with *new*. The world generation system should be responsible for constructing the entire graph to ensure correctness and maintainability.

## Data Pipeline
The class functions as a simple, stateless transformation stage in a larger data processing pipeline.

> Flow:
> Upstream Density Node -> process() returns `double` -> **SqrtDensity** applies `v < 0.0 ? -Math.sqrt(-v) : Math.sqrt(v)` -> returns transformed `double` -> Downstream Consumer (e.g., another Density node or a block placer)

