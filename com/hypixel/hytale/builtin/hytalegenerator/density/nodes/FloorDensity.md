---
description: Architectural reference for FloorDensity
---

# FloorDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class FloorDensity extends Density {
```

## Architecture & Concepts
The FloorDensity class is a fundamental component within the procedural world generation engine. It functions as a **modifier node** within a larger Directed Acyclic Graph (DAG) of Density objects. The entire graph collectively defines a 3D density function, which is ultimately sampled to determine the shape of the world's terrain.

The specific role of FloorDensity is to act as a mathematical clamp, enforcing a minimum value on the data flowing through it. It takes a single Density node as input, processes it to get a value, and returns the greater of either the input's value or its own configured limit.

This node is critical for shaping terrain features such as:
- Creating flat ocean floors or lakebeds by setting a floor at the water level.
- Generating plateaus or mesas by preventing noise functions from carving valleys too deeply.
- Ensuring a minimum ground level across a biome.

Conceptually, it is a simple filter: `output = max(input, limit)`.

### Lifecycle & Ownership
- **Creation:** FloorDensity nodes are not instantiated directly by gameplay code. They are created by the world generator's configuration loader when it parses a world generation preset. These presets define the full density graph for a given world or dimension.
- **Scope:** The lifetime of a FloorDensity instance is tied to the generation of a specific set of world chunks. It is a short-lived, computational object that exists only for the duration of the density calculation phase.
- **Destruction:** The entire density graph, including all its nodes, is dereferenced and becomes eligible for garbage collection after the final terrain for a region has been generated and stored.

## Internal State & Concurrency
- **State:** The internal state consists of the `limit` value and a reference to the `input` Density node. This state is configured during the graph construction phase and is treated as **immutable** during the processing phase. The object itself does not cache results or change its configuration during computation.
- **Thread Safety:** The `process` method is re-entrant and inherently thread-safe. It performs a pure function calculation based on its inputs and contains no locks or mutable instance variables. The world generation system heavily parallelizes chunk generation, and multiple worker threads can safely invoke `process` on a shared FloorDensity instance concurrently.

**WARNING:** While the processing phase is thread-safe, the graph construction phase is not. Modifying the graph via `setInputs` while other threads are processing it will lead to race conditions and undefined behavior. Graph construction must be fully completed before it is handed off to worker threads.

## API Surface
The public contract is minimal, focusing exclusively on graph construction and processing.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(N) | Recursively processes its input node and returns the result, clamped to a minimum value defined by its internal limit. N is the depth of the input graph. |
| setInputs(Density[] inputs) | void | O(1) | Connects this node to its upstream data source. Throws an exception if the input array is malformed. This method is for graph construction only. |

## Integration Patterns

### Standard Usage
FloorDensity is used as a building block within a larger density graph. A generator preset will define a chain of nodes, with FloorDensity used to control the lower bounds of a noise function's output.

```java
// Conceptual graph construction during world-gen initialization
// 1. Create a source noise function
Density perlinNoise = new PerlinNoiseNode(...);

// 2. Create the FloorDensity node to clamp the noise
// This ensures the terrain never goes below a density of -0.5
Density terrainFloor = new FloorDensity(-0.5, perlinNoise);

// 3. Use the clamped result in a downstream node
Density finalTerrain = new CarverNode(terrainFloor, ...);

// The 'finalTerrain' graph is now ready for processing by worker threads.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a FloorDensity node in gameplay logic using `new`. The world generation system is responsible for building the graph from configuration files.
- **Concurrent Modification:** Do not call `setInputs` on a node in a density graph that is actively being processed by generator threads. This will corrupt the graph structure and cause catastrophic generation failures.
- **Graph Cycles:** Creating a dependency where a FloorDensity node is, directly or indirectly, its own input will cause a `StackOverflowError` during the `process` call. The graph builder must validate that the graph is a DAG.

## Data Pipeline
The flow of data through this component is linear and unidirectional. It acts as a simple filter in a larger stream of calculations.

> Flow:
> Upstream Node (e.g., PerlinNoiseNode) -> `process()` -> Raw Density Value -> **FloorDensity.process()** -> `Math.max(Raw Value, limit)` -> Clamped Density Value -> Downstream Node (e.g., CarverNode)

