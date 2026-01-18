---
description: Architectural reference for YOverrideDensity
---

# YOverrideDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class YOverrideDensity extends Density {
```

## Architecture & Concepts
The YOverrideDensity class is a **Coordinate Space Modifier** node within the procedural world generation engine. It operates as a component in a composable graph of Density functions, where each node contributes to the final density value at a given point in 3D space.

The specific function of this node is to intercept a request for a density value and force the evaluation of its child node at a fixed, predefined Y-coordinate. It effectively "flattens" the query space along the vertical axis. Any request to `process` a point (X, Y, Z) will be transformed into a request to its child node at (X, *constant_value*, Z).

This is a fundamental tool for creating horizontally consistent geological features. For example, it can be used to sample a 2D noise function to define a bedrock layer or a water table, ensuring the feature's shape is independent of the query's altitude.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level graph builder, typically during the deserialization of a world generation preset. It is not created by core engine services but is a building block used by generator logic.
- **Scope:** The lifetime of a YOverrideDensity instance is bound to the Density graph it is a part of. It persists as long as that graph is required for a generation task.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It is reclaimed once the parent Density graph is no longer referenced, usually after a world chunk has been fully generated.

## Internal State & Concurrency
- **State:** This class is stateful. It maintains two fields:
    - **input:** A mutable reference to the child Density node to which it delegates processing.
    - **value:** An immutable double representing the fixed Y-coordinate for all child queries.
- **Thread Safety:** This class is **conditionally thread-safe**. The `process` method is safe for concurrent execution by multiple world generation threads, as it only reads internal state and operates on local context objects. However, the `setInputs` method introduces mutability.

    **WARNING:** The density graph structure, including this node's inputs, must be considered immutable during the generation phase. Calling `setInputs` from one thread while other threads are calling `process` will result in a severe race condition and undefined behavior. Graph construction and modification must be completed before it is used by worker threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(C) | Transforms the context's position to use a fixed Y-value and delegates to the child input node. C is the complexity of the child's process method. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the child input node. Throws AssertionError if the input array does not contain exactly one element. |

## Integration Patterns

### Standard Usage
This node is used to wrap another Density function, effectively projecting its output onto a horizontal plane. This is common for creating strata or sampling 2D patterns in a 3D context.

```java
// Assume noiseSource is a 3D Perlin noise density function
Density noiseSource = new PerlinNoiseDensity(...);

// Create a bedrock layer by sampling the 3D noise, but only at Y = 5.0
// This makes the "bedrock" pattern consistent regardless of altitude.
double bedrockLevel = 5.0;
Density bedrockDensity = new YOverrideDensity(noiseSource, bedrockLevel);

// When bedrockDensity.process is called for any Y, it will always query
// noiseSource at Y = 5.0.
double valueAtSurface = bedrockDensity.process(new Context(new Vector3d(10, 64, 20)));
double valueUnderground = bedrockDensity.process(new Context(new Vector3d(10, 10, 20)));

// valueAtSurface will be equal to valueUnderground.
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call `setInputs` on a YOverrideDensity node that is part of a density graph actively being used for world generation. All graph wiring must be completed before processing begins.
- **Incorrect Input Configuration:** Do not call `setInputs` with an array that does not contain exactly one Density object. The node is a single-input transformer and cannot function otherwise. This will trigger an assertion in development environments.

## Data Pipeline
The primary flow involves the transformation of the `Density.Context` object before it is passed down the graph.

> Flow:
> `process(originalContext)` → Create `modifiedPosition` with Y set to `this.value` → Create `childContext` with `modifiedPosition` → **YOverrideDensity** delegates to `this.input.process(childContext)` → Return `double` value from child.

