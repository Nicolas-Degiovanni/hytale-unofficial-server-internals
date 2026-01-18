---
description: Architectural reference for MaxDensity
---

# MaxDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class MaxDensity extends Density {
```

## Architecture & Concepts
The MaxDensity class is a fundamental *Composite Node* within the procedural world generation system. It operates as a component in a larger directed acyclic graph (DAG) of Density functions, which collectively define the shape and substance of the game world.

Architecturally, this class implements the Composite design pattern. It is a Density function whose behavior is defined by a collection of other child Density functions. Its specific role is to act as an aggregator, evaluating all of its input densities and returning the single highest value.

In the context of procedural generation and Constructive Solid Geometry (CSG), the max operation is equivalent to a boolean **Union**. When combining two or more density fields (e.g., spheres, planes), taking the maximum value effectively merges their volumes, creating a single contiguous shape. This makes MaxDensity a critical building block for blending terrain features, merging biomes, or combining geometric primitives into complex structures.

### Lifecycle & Ownership
- **Creation:** MaxDensity nodes are not created dynamically during the game loop. They are instantiated once during the loading and parsing of a world generation configuration, typically from a JSON or similar data-driven definition. A graph builder or configuration factory is responsible for its instantiation.
- **Scope:** The object's lifetime is bound to the currently active world generator configuration. It persists in memory as part of the generator's definition graph.
- **Destruction:** The object is eligible for garbage collection when the world generator configuration is unloaded or replaced, for instance, when a player leaves a world or a server changes its active world.

## Internal State & Concurrency
- **State:** The primary state of a MaxDensity instance is its `inputs` array. This field is mutable via the public `setInputs` method, making the object's structure inherently mutable. During a `process` call, however, the calculation itself is stateless, operating only on local stack variables.

- **Thread Safety:** This class is **not thread-safe** by default. The `process` method is re-entrant and safe to call from multiple threads *only if* the `inputs` array is not modified concurrently.
    - **WARNING:** Calling `setInputs` or directly modifying the public `inputs` field from one thread while another thread is executing the `process` method will lead to undefined behavior, including potential `ArrayIndexOutOfBoundsException` or incorrect density calculations.
    - All mutation of the density graph, including calls to `setInputs`, must be completed before the world generator is used by worker threads.

## API Surface
The public API provides the core functionality for processing and configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MaxDensity(List<Density> inputs) | constructor | O(N) | Constructs the node, converting the input List to an internal array. |
| process(Density.Context context) | double | O(N) | Evaluates all N child nodes and returns the maximum computed density value. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the internal array of child nodes. This is not a thread-safe operation. |

## Integration Patterns

### Standard Usage
MaxDensity is intended to be used as a node within a larger, pre-constructed density graph. It is configured once and then processed many times by the world generator.

```java
// Assume sphere1 and sphere2 are existing Density objects
List<Density> shapesToUnion = new ArrayList<>();
shapesToUnion.add(sphere1);
shapesToUnion.add(sphere2);

// Create a node that represents the union of the two spheres
MaxDensity union = new MaxDensity(shapesToUnion);

// This 'union' node is now passed to other parts of the
// world generator to be processed for a given coordinate.
double finalDensity = union.process(densityContext);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call `setInputs` on a MaxDensity node that is actively being used by the world generator's worker threads. The graph should be treated as immutable after its initial construction.
- **External Array Mutation:** The constructor copies the input list into a new array. However, if `setInputs` is used, the provided array is assigned by reference. Modifying this external array after the call will dangerously alter the internal state of the MaxDensity node.
- **Empty Inputs:** While the implementation safely returns 0.0 for an empty input list, this often indicates a configuration error. A MaxDensity node with no inputs serves no purpose in the generation graph and should be avoided.

## Data Pipeline
The component acts as a many-to-one function in the data pipeline, reducing multiple density values into a single one.

> Flow:
> Array of child Density nodes -> **MaxDensity.process()** -> Iterative evaluation of each child -> Selection of maximum value -> Single `double` output

