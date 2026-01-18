---
description: Architectural reference for SmoothMinDensity
---

# SmoothMinDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Component Node

## Definition
```java
// Signature
public class SmoothMinDensity extends Density {
```

## Architecture & Concepts
The SmoothMinDensity class is a specialized processing node within the procedural world generation's *Density Graph*. It functions as a mathematical combiner, taking two input density fields and blending them using a smooth minimum function. This is a fundamental operation for creating organic, natural-looking terrain features.

Unlike a standard `Math.min(a, b)` which produces a sharp, angular intersection, SmoothMinDensity uses a polynomial interpolation controlled by the `range` parameter. This creates a soft, curved transition where two shapes or material densities meet, essential for features like cave entrances, river banks, and smooth overhangs.

Within the larger system, this class acts as an intermediate node in a directed acyclic graph (DAG) of Density objects. Each node in this graph represents a step in the terrain generation algorithm—from basic noise functions to complex transformations. SmoothMinDensity is one of the primary tools for combining these steps into a cohesive final output.

## Lifecycle & Ownership
- **Creation:** Instances are created during the initialization phase of the world generator. They are typically instantiated by a graph-building service that parses a world generation configuration file (e.g., a JSON preset). Direct instantiation during the game loop is strongly discouraged.
- **Scope:** The object's lifetime is bound to the Density Graph it is a part of. It persists as long as the world generator configuration is active.
- **Destruction:** The object is eligible for garbage collection when the entire Density Graph is dereferenced and replaced, for example, when a server changes worlds or shuts down. There are no explicit destruction methods.

## Internal State & Concurrency
- **State:** The object contains a mix of immutable and mutable state. The `range` field is immutable, set at construction and defining the smoothness of the blend. The `inputA` and `inputB` fields are mutable references to other Density nodes and can be rewired after construction via the `setInputs` method.

- **Thread Safety:** **This class is not thread-safe.** The internal state, specifically the `inputA` and `inputB` references, can be modified at any time.

    **WARNING:** Modifying the inputs via `setInputs` while another thread is executing the `process` method will lead to race conditions, non-deterministic output, and potential NullPointerExceptions. The owning world generation system MUST ensure that the structure of the Density Graph is static during the processing of a world chunk. All graph modifications must be completed before parallel generation tasks begin.

## API Surface
The public contract is minimal, focusing on graph construction and processing.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(D) | Recursively processes its input nodes and returns the blended density value. D is the depth of the sub-graph. Returns 0.0 if inputs are not configured. |
| setInputs(Density[] inputs) | void | O(1) | Wires this node to its inputs. Throws ArrayIndexOutOfBoundsException if the array is null or empty. |

## Integration Patterns

### Standard Usage
This node is never used in isolation. It is always part of a larger Density Graph constructed by the world generator. The `process` method is invoked by a parent node or the graph's root executor, not by user code.

```java
// Conceptual example of graph construction
// This code would exist within a world generator setup routine.

// Create two source density fields
Density sphere = new SphereDensity(radius);
Density plane = new PlaneDensity(height);

// Create the SmoothMin node to blend them
double blendRange = 8.0;
SmoothMinDensity blendNode = new SmoothMinDensity(blendRange, sphere, plane);

// The blendNode is now ready to be used as an input for another node
// or as the root of the graph.
```

### Anti-Patterns (Do NOT do this)
- **State Mutation During Processing:** Never call `setInputs` on a node that is part of a graph currently being processed by world generation threads. This will break generation integrity.
- **Relying on Null Fallback:** The `process` method returns 0.0 if its inputs are null. Code should not rely on this behavior for logic. A well-formed Density Graph should be validated upon creation to ensure all required inputs are connected.
- **Direct Instantiation:** While technically possible, creating new instances of this class outside of the world generator's initialization phase is an anti-pattern that can lead to severe performance degradation and memory churn.

## Data Pipeline
SmoothMinDensity acts as a transformation stage in the density calculation pipeline. It consumes data from two upstream nodes and provides a single output to a downstream node.

> Flow:
> Upstream Node A.process() ➞ `double valueA` ↘
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; **SmoothMinDensity.process()** ➞ `double finalValue` ➞ Downstream Node
> Upstream Node B.process() ➞ `double valueB` ↗

