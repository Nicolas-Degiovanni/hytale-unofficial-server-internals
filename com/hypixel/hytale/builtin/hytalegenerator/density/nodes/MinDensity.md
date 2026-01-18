---
description: Architectural reference for MinDensity
---

# MinDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class MinDensity extends Density {
```

## Architecture & Concepts
The MinDensity class is a composite node within the procedural world generation framework. It functions as a mathematical aggregator in a directed acyclic graph of Density objects. Its primary role is to process a collection of input Density nodes and return the single lowest (minimum) value among them.

This component is a fundamental building block for blending and shaping terrain. For example, it can be used to carve a river canyon into a mountain range by taking the minimum density value between the mountain function and the inverted canyon function. It effectively acts as a logical AND operation in constructive solid geometry, selecting the most intrusive feature at any given point in 3D space.

## Lifecycle & Ownership
- **Creation:** MinDensity nodes are not instantiated directly by the main game loop. They are constructed by a higher-level system, typically a world generator or a graph parser, which assembles the complete density graph from a configuration file or procedural logic.
- **Scope:** The lifetime of a MinDensity instance is bound to the lifecycle of the parent density graph it is a part of. It persists as long as that specific graph is required for generating world chunks.
- **Destruction:** The object is eligible for garbage collection once the parent density graph is no longer referenced. This typically occurs when the world generation for a specific region is complete and its configuration is unloaded.

## Internal State & Concurrency
- **State:** Mutable. The core state is the internal `inputs` array, which holds references to other Density nodes. This array can be replaced at runtime via the `setInputs` method, allowing for dynamic reconfiguration of the density graph.
- **Thread Safety:** **This class is not thread-safe.** The `inputs` field is accessed without any synchronization. If one thread calls `setInputs` while another thread is iterating through the array in the `process` method, a `ConcurrentModificationException` is possible, or the calculation will proceed with an inconsistent set of inputs, leading to severe visual artifacts or crashes. All access and modification must be externally synchronized, typically by ensuring that a single world generation worker thread operates on a given graph at one time.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(N) | Calculates the minimum density value from all child nodes. N is the number of direct inputs. Returns 0.0 if no inputs are present. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the internal list of input nodes. This is a highly sensitive operation. See Concurrency section. |

## Integration Patterns

### Standard Usage
MinDensity is used to combine multiple density sources, selecting the lowest value to form a final shape.

```java
// Assume noise1 and noise2 are other initialized Density objects
List<Density> sources = new ArrayList<>();
sources.add(noise1);
sources.add(caveSystemDensity);

// Create the node with its inputs
MinDensity finalShape = new MinDensity(sources);

// Process the node to get the final value for a world coordinate
// The context object contains the x,y,z coordinates being sampled
double value = finalShape.process(context);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call `setInputs` from one thread while `process` may be executing on another. This will corrupt the state of the generation pipeline. The graph structure should be considered immutable during processing.
- **Empty Inputs:** While the implementation defaults to returning 0.0 for an empty input list, this is almost always a sign of a configuration error in the world generator. An empty MinDensity node serves no purpose and can lead to unexpected flat or zero-density regions.

## Data Pipeline
MinDensity acts as a funnel within the larger density calculation pipeline, reducing multiple data streams into a single output value.

> Flow:
> World Generator -> Assembles Density Graph -> **MinDensity Node** receives multiple input Density values -> **MinDensity.process()** iterates and compares values -> A single minimum `double` is returned -> Value contributes to final terrain shape

