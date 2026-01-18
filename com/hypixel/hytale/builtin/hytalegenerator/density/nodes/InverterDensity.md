---
description: Architectural reference for InverterDensity
---

# InverterDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class InverterDensity extends Density {
```

## Architecture & Concepts
The InverterDensity class is a foundational component within the procedural world generation engine, operating as a node in a composable *Density Graph*. Its sole function is to perform arithmetic negation on the output of another Density node.

Architecturally, this class implements a form of the Decorator pattern. It wraps a single child Density component, conforming to the same base Density interface, but modifies the output before returning it. This allows for the construction of complex procedural logic by chaining together simple, single-purpose nodes. For example, a function that generates positive density values for mountains can be passed into an InverterDensity to create inverse values representing valleys or caves, without altering the original function.

This node is a unary operator, meaning it accepts exactly one input and produces one output. It is a fundamental building block for manipulating terrain heightmaps and 3D density fields.

### Lifecycle & Ownership
- **Creation:** InverterDensity nodes are instantiated by a higher-level graph construction service, typically when parsing a world generation preset from a configuration file. They are not intended to be created directly during the main game loop.
- **Scope:** The lifetime of an InverterDensity instance is strictly bound to its parent Density Graph. It persists in memory only as long as the graph is required for a generation task.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection once the parent Density Graph is released. This typically occurs after a world chunk or region has been fully generated and its voxel data has been computed.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The primary state is the `input` field, which holds a reference to the child Density node. This reference can be dynamically changed after instantiation via the `setInputs` method.

- **Thread Safety:** This class is **not thread-safe**. Concurrent calls to `setInputs` and `process` will lead to race conditions and unpredictable behavior. The entire Density Graph is designed for single-threaded execution within a dedicated world generation worker thread. All modifications to the graph structure must be completed before it is used for processing.

## API Surface
The public API provides the core functionality for processing data and configuring the node's input.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(N) | Processes the input node and returns its arithmetically negated result. Complexity is dependent on the entire subgraph of the input node. Returns 0.0 if no input is configured. |
| setInputs(Density[] inputs) | void | O(1) | Reconfigures the node's input source. **WARNING:** This is a state-mutating operation. It discards the current input and replaces it with the first element of the provided array. |

## Integration Patterns

### Standard Usage
The InverterDensity is used to wrap another Density node, effectively inverting its output within a larger generation graph.

```java
// Assume 'mountainDensity' is a pre-existing node that produces positive values
Density mountainDensity = new PerlinNoiseDensity(...);

// Create an inverter to turn mountains into valleys
Density valleyDensity = new InverterDensity(mountainDensity);

// The graph processor will later invoke process()
// The result will be the negative of the mountainDensity output
double densityValue = valleyDensity.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation During Processing:** Never call `setInputs` on a node that is part of a Density Graph currently being evaluated. The graph's structure must be considered immutable during the `process` phase to prevent non-deterministic output.
- **Direct Instantiation in Game Logic:** Do not use `new InverterDensity()` in general game systems. These nodes should only be constructed during the world generator's initialization phase.
- **Null Input Processing:** While the `process` method is null-safe and returns 0.0, relying on this behavior can mask configuration errors in the Density Graph. All nodes should be properly connected before processing.

## Data Pipeline
The InverterDensity acts as a simple transformation step in the larger density calculation pipeline.

> Flow:
> World Generator -> Density Graph Traversal -> **InverterDensity.process()** -> Invokes `input.process()` -> Receives positive/negative `double` from child -> Returns negated `double` -> Voxel Material Selection Logic

