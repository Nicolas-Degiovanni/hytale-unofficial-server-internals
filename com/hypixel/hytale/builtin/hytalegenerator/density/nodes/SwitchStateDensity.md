---
description: Architectural reference for SwitchStateDensity
---

# SwitchStateDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class SwitchStateDensity extends Density {
```

## Architecture & Concepts
The SwitchStateDensity class is a structural node within the procedural world generation's Density Graph system. Its primary architectural role is to modify the execution context for a subgraph of density nodes. It acts as a state injector, allowing a portion of the generation graph to operate with a specific, overridden configuration value.

Conceptually, this node takes a single child Density node as input. When the graph is processed, SwitchStateDensity creates a new, temporary Density.Context, injects its configured integer `switchState` into this new context, and then processes its child node using this modified context. This allows downstream nodes, particularly conditional nodes like SwitchDensity, to alter their behavior based on the state provided by this ancestor.

**WARNING:** A critical review of the `process` method reveals a potential implementation anomaly. The method creates a new `childContext` and sets the `switchState`, but then proceeds to call `this.input.process` with the original, unmodified `context`. This effectively renders the node a pass-through that does not propagate the new state. This behavior must be accounted for when debugging world generation logic, as it deviates from the apparent design intent.

## Lifecycle & Ownership
- **Creation:** Instantiated by a graph-building service during the parsing of world generation zone files. It is not intended for manual instantiation by developers. Each instance represents a single, configured node in a specific Density Graph.
- **Scope:** The object's lifetime is bound to the Density Graph it is a part of. It persists for the duration of a single, self-contained generation task, such as generating the density values for a world chunk.
- **Destruction:** Marked for garbage collection once the generation task is complete and the root of the Density Graph is no longer referenced.

## Internal State & Concurrency
- **State:** This class is stateful. It holds a reference to its child `input` node and an immutable `switchState` value set at construction. The `input` field is mutable and can be reassigned via the `setInputs` method during the graph assembly phase.
- **Thread Safety:** This class is **not thread-safe**. The Density Graph processing is designed for single-threaded execution within a given generation task. Concurrent calls to `process` or modification of the `input` field via `setInputs` during processing will result in unpredictable behavior and data corruption. All graph manipulation must be completed before processing begins.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(N) | Recursively processes the child density node. N is the number of nodes in the subgraph. **Warning:** Does not correctly propagate the modified context to its child. |
| setInputs(Density[] inputs) | void | O(1) | Sets the single child node for this operator. Expects an array with exactly one element for normal operation. |

## Integration Patterns

### Standard Usage
This node is not used directly in Java code but is defined declaratively within world generation configuration files. The system's graph parser instantiates and connects it. The conceptual usage involves wrapping a subgraph to control its execution environment.

```java
// Conceptual representation of a graph
// This code is for illustration; direct instantiation is an anti-pattern.

// A subgraph that should operate under state 5
Density subGraph = new SomeOtherDensityNode(...);

// Wrap the subgraph with a SwitchStateDensity node
Density statefulGraph = new SwitchStateDensity(subGraph, 5);

// When the parent graph is processed, it will call:
// double result = statefulGraph.process(initialContext);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never construct a SwitchStateDensity using `new`. The world generator's graph factory is responsible for creating and wiring nodes from configuration. Manual creation bypasses asset tracking and validation.
- **Assuming State Propagation:** Do not rely on this node to correctly pass the `switchState` to its children. Due to the implementation of the `process` method, the child node will receive the original, unmodified context. Any logic dependent on this state change will fail.
- **Multiple Inputs:** The `setInputs` method will only ever consume the first element of the provided array. Providing more than one input has no effect and indicates a misconfigured graph.

## Data Pipeline
The intended data flow for this component involves the modification of the execution context.

> Flow:
> Parent Node Process -> **SwitchStateDensity.process(context)** -> New `childContext` created with `switchState` -> Child Node Process is called with **original `context`** -> Result returned up the call stack.

The deviation from the expected pipeline is the critical point. The `childContext` is created but not used in the subsequent call, breaking the state propagation chain.

