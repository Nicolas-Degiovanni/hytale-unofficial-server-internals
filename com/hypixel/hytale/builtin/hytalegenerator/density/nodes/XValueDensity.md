---
description: Architectural reference for XValueDensity
---

# XValueDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class XValueDensity extends Density {
```

## Architecture & Concepts
The XValueDensity class is a foundational node type within the procedural world generation system, specifically operating within a Density Graph. A Density Graph is a network of nodes that collectively compute a scalar value (density) for any given point in 3D space, which in turn defines the shape of terrain, cave systems, and other geological features.

This class serves as a primitive **Input Node**. Its sole responsibility is to provide the world-space X-coordinate of the point currently being evaluated. It acts as a bridge between the absolute coordinate system of the world generator and the relative computational graph. By injecting the X-coordinate, it enables subsequent nodes in the graph to create patterns, gradients, and features that are dependent on their east-west position in the world.

It is one of the most fundamental building blocks for creating non-uniform, spatially-aware procedural content.

## Lifecycle & Ownership
- **Creation:** Instances of XValueDensity are not intended for manual creation. They are instantiated by a higher-level system, typically a `DensityGraphLoader` or a similar parser, which constructs the entire density graph from a data-driven configuration file (e.g., JSON).
- **Scope:** The lifetime of an XValueDensity object is strictly bound to the lifetime of the parent Density Graph it is a part of. It persists as long as the world generator configuration is active.
- **Destruction:** The object is eligible for garbage collection when its parent Density Graph is unloaded or replaced, for instance, when a world is closed or the server shuts down. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is exclusively determined by the `Density.Context` object passed into its `process` method. It performs no caching.

- **Thread Safety:** XValueDensity is inherently **thread-safe**. Due to its complete lack of internal state, the `process` method is a pure function. Multiple threads from the world generation worker pool can call it concurrently without any risk of race conditions or data corruption. This design is critical for achieving high-performance, parallelized chunk generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) | Returns the X-coordinate from the provided context. Throws NullPointerException if the context or its position is null. |
| setInputs(Density[] inputs) | void | O(1) | No-op. This node does not accept inputs; any provided array is ignored. This fulfills the contract of the parent Density class. |

## Integration Patterns

### Standard Usage
This class is not used directly in procedural code. Instead, it is declared as a node within a world generation asset file. The engine's density graph evaluator then invokes its `process` method during terrain calculation.

A developer interacting with a pre-built graph might invoke it as follows:
```java
// Assume 'densityGraph' is a fully constructed graph and 'context' is valid
// for a specific world location. This node would be one of many in the graph.

// This call simulates the engine's evaluation of this specific node.
double xCoordinate = anXValueDensityNode.process(context);

// The returned value is then used by other nodes.
// For example, a noise node might take this as an input.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new XValueDensity()`. These nodes must only be created as part of a graph definition loaded by the engine to ensure proper integration.
- **Meaningless Input:** Calling `setInputs` has no effect. The design of this node is to be a graph origin, not a processor of other nodes' outputs.
- **Context-less Invocation:** The `process` method is meaningless without a valid `Density.Context` supplied by the world generation engine for a specific coordinate.

## Data Pipeline
The flow of data through this component is simple and linear. It acts as a source of information for the rest of the density graph.

> Flow:
> World Generator Iteration -> `Density.Context` Creation -> **XValueDensity.process(context)** -> `double` (x-coordinate) -> Upstream Consumer Node (e.g., AddDensity, PerlinNoiseDensity)

