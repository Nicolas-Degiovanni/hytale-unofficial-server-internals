---
description: Architectural reference for ZOverrideDensity
---

# ZOverrideDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Component

## Definition
```java
// Signature
public class ZOverrideDensity extends Density {
```

## Architecture & Concepts
The ZOverrideDensity class is a structural node within the procedural world generation's density graph. It functions as a coordinate space manipulator, specifically designed to intercept a density query and override its Z-coordinate with a fixed value before passing the query to a child Density node.

This component is fundamental for projecting 3D density functions onto a 2D plane. For instance, it allows a 3D Perlin noise function to be sampled as if it were a 2D heightmap by forcing all samples to occur at a constant Z-level. It acts as a decorator or an adapter in a composable graph, modifying the input for its wrapped `input` Density without altering the child's internal logic.

Its role is to decouple density functions from the specific spatial context in which they are evaluated, enabling the reuse of complex 3D functions for 2D or layered generation tasks.

### Lifecycle & Ownership
- **Creation:** ZOverrideDensity nodes are not instantiated directly by game systems. They are created by a world generator's configuration loader, which parses a declarative graph definition (e.g., from a JSON file) and constructs the corresponding network of Density objects.
- **Scope:** The object's lifetime is bound to the active world generator configuration. It persists as long as the generator is in use for a given world or dimension.
- **Destruction:** The object is eligible for garbage collection when the world generator configuration is unloaded or replaced. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** The class is stateful, containing references to its child `input` node and its configured `value`. However, after initial configuration, its state is effectively immutable during a generation pass. The `setInputs` method allows for the graph to be rewired, but this is a configuration-time operation, not a runtime one.
- **Thread Safety:** This class is **conditionally thread-safe**. The `process` method is re-entrant and does not mutate the instance's state. It allocates new Context and Vector3d objects for each call, preventing data races within this class. Concurrency safety is therefore dependent on the thread safety of the downstream `input` Density node.

    **WARNING:** If the provided `input` Density is not thread-safe, the entire generation pipeline can be compromised. All nodes in a density graph intended for parallel execution must be re-entrant.

## API Surface
The public API is minimal, focusing exclusively on graph construction and processing.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ZOverrideDensity(input, value) | constructor | O(1) | Constructs the node with its child and the fixed Z value. |
| process(context) | double | O(1) + O(input) | Overrides the Z-coordinate in the context and invokes the child `input` node. Complexity is dominated by the child's process call. |
| setInputs(inputs) | void | O(1) | Replaces the child node. Throws an AssertionError if the input array size is not exactly 1. |

## Integration Patterns

### Standard Usage
This node is used to wrap another Density function to sample it on a horizontal plane. This is a common pattern for generating terrain heightmaps from 3D noise.

```java
// Assume 'threeDimensionalNoise' is a pre-existing Density object
// and 'context' is the service provider for the generator.

// Create a node that will always sample the 3D noise at Z = 50.0
double fixedZLevel = 50.0;
Density heightmapSampler = new ZOverrideDensity(threeDimensionalNoise, fixedZLevel);

// When process is called, the original Z-coordinate from the queryContext
// will be ignored and replaced with 50.0.
double densityValue = heightmapSampler.process(queryContext);
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Input Configuration:** The `setInputs` method is rigid. Calling it with an array whose length is not equal to one will cause a fatal assertion error and crash the generator. This is by design to enforce graph integrity.
- **Dynamic Rewiring:** Do not call `setInputs` during a generation pass. The density graph is expected to be static while processing a world chunk. Modifying the graph mid-generation can lead to undefined behavior and visual seams in the world.
- **Ignoring Child Complexity:** While ZOverrideDensity itself is a low-cost operation, it provides no performance benefits if the wrapped `input` node is computationally expensive. Its purpose is purely logical, not optimization.

## Data Pipeline
The primary function of this class is to transform the `Density.Context` object before it reaches the next node in the graph.

> Flow:
> World Generator Query -> Density.Context (X, Y, Z) -> **ZOverrideDensity** -> New Density.Context (X, Y, fixed value) -> Input Density Node -> Final double value

