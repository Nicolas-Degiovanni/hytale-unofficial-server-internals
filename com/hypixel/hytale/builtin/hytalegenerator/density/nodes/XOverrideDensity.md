---
description: Architectural reference for XOverrideDensity
---

# XOverrideDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class XOverrideDensity extends Density {
```

## Architecture & Concepts
The XOverrideDensity class is a coordinate transformation node within the procedural world generation's density graph. Its primary function is to intercept a sampling request, discard the incoming X-coordinate, and substitute it with a fixed, predefined value before passing the request to its child node.

This effectively samples a 2D slice (a Y-Z plane) of the child density function at a specific X-coordinate. It is a form of *domain warping* used to create geological features or structures that are constant along a single axis. For example, it could be used to define a perfectly flat, vertical wall or a biome boundary that runs infinitely along the X-axis.

As a node in a directed acyclic graph (DAG) of Density functions, it acts as a modifier, altering the input to another, more complex density calculation.

### Lifecycle & Ownership
- **Creation:** Instantiated by the world generator's configuration loader during the setup phase. These nodes are typically defined in data files (e.g., JSON or HOCON) and are assembled into a complete density graph before any world generation begins.
- **Scope:** The object's lifetime is bound to the parent world generator instance. It persists as long as that specific generator configuration is active.
- **Destruction:** The node is eligible for garbage collection when the entire density graph is discarded. This typically occurs when the server shuts down or a new world with a different generator configuration is loaded.

## Internal State & Concurrency
- **State:** This class is stateful. It maintains two fields: a reference to its child `input` Density node and the constant `value` for the X-coordinate. The `input` field is mutable and can be changed post-construction via the `setInputs` method, allowing for dynamic graph reconfiguration.
- **Thread Safety:** The `process` method is re-entrant and safe for concurrent execution by multiple world generation worker threads. It operates on a local copy of the context and does not modify its own internal state during processing.

    **WARNING:** The class as a whole is **not thread-safe**. A race condition will occur if one thread calls `setInputs` while another thread is executing `process`. The density graph is expected to be fully constructed and considered immutable *before* being used by the multi-threaded generation engine.

## API Surface
The public contract is designed for two distinct phases: graph construction (`setInputs`) and world generation (`process`).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(C) | Samples the child density function at a modified coordinate. Complexity is dependent on the child node (C). |
| setInputs(Density[] inputs) | void | O(1) | Wires this node's input to another Density node. Throws AssertionError if the input array size is not exactly 1. |

## Integration Patterns

### Standard Usage
This node is used to wrap another Density function, forcing all samples to occur on a specific X-plane.

```java
// Assume 'perlin' is a pre-existing PerlinNoiseDensity node
// and 'context' is the service provider for the generator.

// Create a node that will always sample the perlin noise at X=50.0
Density fixedPlaneNoise = new XOverrideDensity(perlin, 50.0);

// During generation, the original x-coordinate (e.g., 123.4) is ignored
double densityValue = fixedPlaneNoise.process(new Density.Context(123.4, 10.0, 20.0));
// The 'perlin' node was actually sampled at (50.0, 10.0, 20.0)
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not call `setInputs` on a node while the density graph is being actively processed by worker threads. All graph wiring must be completed during the single-threaded initialization phase.
- **Invalid Input Array:** Passing an array to `setInputs` with a length other than 1 will cause a hard crash via an assertion error. This indicates a misconfigured generator graph.

## Data Pipeline
XOverrideDensity acts as a simple but powerful transformation filter in the data pipeline. It modifies the spatial context of a request before it continues down the graph.

> Flow:
> Density.Context (Original Position) -> **XOverrideDensity** (Creates new Context with X = `value`) -> Child `input` Node -> `double` (Density Value)

