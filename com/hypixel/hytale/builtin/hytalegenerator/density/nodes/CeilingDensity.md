---
description: Architectural reference for CeilingDensity
---

# CeilingDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient / Data Node

## Definition
```java
// Signature
public class CeilingDensity extends Density {
```

## Architecture & Concepts
The CeilingDensity class is a fundamental component within the procedural world generation framework. It functions as a *filter node* in a directed acyclic graph (DAG) of density calculations. Its sole purpose is to enforce an upper bound, or a "ceiling", on a density value provided by an input node.

In the context of terrain generation, density values often map to material presence or terrain height. By applying a CeilingDensity, a world designer can flatten plateaus, create underwater seabeds at a consistent depth, or cap the height of specific geological features. It is a non-destructive clamping operation, taking a stream of density values and ensuring none exceed a predefined limit.

This class embodies the composition pattern, where complex generation logic is built by chaining together simple, single-purpose nodes. It is designed to be lightweight and computationally inexpensive, making it suitable for use in deeply nested generation graphs.

## Lifecycle & Ownership
- **Creation:** CeilingDensity instances are not managed by a central service registry. They are instantiated dynamically by a higher-level system, typically a *World Generator* or a *Graph Assembler*, which parses a world generation configuration. The constructor requires the limit and an initial input Density.
- **Scope:** The lifetime of a CeilingDensity node is tied to the lifetime of the generation graph it belongs to. It is typically scoped to a single, self-contained generation task, such as the generation of a world chunk column.
- **Destruction:** Once the generation task is complete and the final output (e.g., block data) has been produced, the entire graph of Density nodes, including this instance, becomes unreachable and is reclaimed by the Java garbage collector. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state consists of two fields: the double *limit* and the Density *input*. The *limit* is immutable after construction. The *input* field is mutable via the setInputs method, but this is exclusively intended for use during the initial graph assembly phase. During the processing phase, the object's state should be considered effectively immutable.
- **Thread Safety:** The process method is **conditionally thread-safe**. It performs a read-only operation on its internal state and the provided Context object. Concurrently calling process from multiple threads is safe, provided that the input Density node and the shared Context object are also thread-safe. This design is critical for enabling parallelized chunk generation, where multiple threads can process different coordinates using the same shared generator graph.

**WARNING:** Modifying the input node via setInputs while other threads are executing the process method will lead to race conditions and non-deterministic output. All graph mutations must be completed before the generation process begins.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CeilingDensity(limit, input) | constructor | O(1) | Constructs the node with a hard upper limit and an initial input node. |
| process(context) | double | O(N) | Calculates the density. Returns the lesser of the input node's density or the configured limit. Complexity is O(N) where N is the number of nodes in the input sub-graph. |
| setInputs(inputs) | void | O(1) | Replaces the current input node with the first element from the provided array. Throws ArrayIndexOutOfBoundsException if the array is empty. |

## Integration Patterns

### Standard Usage
CeilingDensity is used to cap the output of another Density node. It is chained together with other nodes to form a processing pipeline.

```java
// Assume a NoiseDensity node that produces values from -1.0 to 1.0
Density noiseSource = new NoiseDensity(seed, frequency);

// Create a CeilingDensity to cap the noise output at 0.5
// This effectively flattens any terrain features above this value.
double plateauLevel = 0.5;
Density cappedNoise = new CeilingDensity(plateauLevel, noiseSource);

// The cappedNoise can now be used by subsequent stages of world generation
double finalValue = cappedNoise.process(context); // Will never be > 0.5
```

### Anti-Patterns (Do NOT do this)
- **Graph Mutation During Processing:** Never call setInputs on a node after the generation process has begun. The graph structure must be considered immutable once processing starts to ensure deterministic and thread-safe behavior.
- **Null Input:** While the code handles a null input by returning 0.0, this is typically an indication of a misconfigured generator graph. A CeilingDensity without an input serves no purpose and should be avoided.

## Data Pipeline
The data flow is a simple filter operation. An incoming density value is compared against a static limit, and the smaller of the two is passed downstream.

> Flow:
> Upstream Density Node -> process(context) -> Input Value -> **CeilingDensity** -> Math.min(Input Value, this.limit) -> Output Value -> Downstream Consumer

