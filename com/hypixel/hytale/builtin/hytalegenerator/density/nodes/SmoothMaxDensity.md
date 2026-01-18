---
description: Architectural reference for SmoothMaxDensity
---

# SmoothMaxDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class SmoothMaxDensity extends Density {
```

## Architecture & Concepts
The SmoothMaxDensity class is a fundamental computational node within the procedural world generation framework. It operates as a mathematical function within a larger, interconnected graph of Density objects. Its primary role is to combine two input density fields, producing a single output value.

Unlike a standard maximum function which results in sharp, angular transitions, SmoothMaxDensity uses a smoothing algorithm to create a soft, organic blend between the two inputs. The degree of this blending is controlled by the `range` parameter. This makes it an essential tool for generating natural-looking terrain features, such as the smooth merging of hills or the gentle slopes around cave entrances.

Architecturally, it is a leaf or branch in a directed acyclic graph (DAG) where each node represents a density calculation. The final terrain is the result of evaluating this entire graph at each point in 3D space.

### Lifecycle & Ownership
- **Creation:** Instances are created by the world generator's graph builder during the initialization phase. This process typically involves parsing a generator configuration file and instantiating the corresponding Density nodes to construct the full calculation graph.
- **Scope:** The object's lifetime is bound to the specific world generator instance it is a part of. It is not a global or session-wide object.
- **Destruction:** The object is eligible for garbage collection when the world generator and its associated graph are discarded. This occurs, for example, when the server shuts down or a new world with a different configuration is loaded.

## Internal State & Concurrency
- **State:** The object holds a mutable state. While the `range` field is final and set at construction, the `inputA` and `inputB` references can be modified post-construction via the `setInputs` method. The class does not cache the results of its `process` method; calculations are performed on every invocation.

- **Thread Safety:** This class is **not thread-safe**.
    - The `process` method is safe for concurrent reads, assuming its inputs (`inputA` and `inputB`) are also safe for concurrent processing. It does not modify its own state during execution.
    - The `setInputs` method is a write operation. Invoking `setInputs` while another thread is executing `process` will create a severe race condition, potentially leading to NullPointerExceptions or inconsistent density calculations.
    - **WARNING:** The entire Density graph must be treated as immutable after its initial construction and before it is used by worker threads for world generation. Modifying the graph structure during processing will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SmoothMaxDensity(range, inputA, inputB) | constructor | O(1) | Constructs a new node with a smoothing range and two initial density inputs. |
| process(Context context) | double | O(N) | Calculates the smoothed maximum of its two inputs. Complexity is dependent on the `process` complexity of its inputs. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the node's inputs. **WARNING:** Not thread-safe. Throws no error if fewer than two inputs are provided, resulting in silent failure. |

## Integration Patterns

### Standard Usage
This node is used to combine two other density functions. The developer assembles a graph of these nodes, which is then processed by the generator.

```java
// Assume PerlinNoiseDensity and ConstantDensity are other Density types
Density perlinField = new PerlinNoiseDensity(seed, scale);
Density groundPlane = new ConstantDensity(-10.0);

// Create a smooth blend between the noise field and the ground plane
double smoothingRange = 8.0;
Density finalTerrain = new SmoothMaxDensity(smoothingRange, perlinField, groundPlane);

// The generator would then call this for each point in the world
// double value = finalTerrain.process(context);
```

### Anti-Patterns (Do NOT do this)
- **Graph Mutation During Generation:** Never call `setInputs` on any node in a Density graph after the world generation process has started. The graph must be fully constructed and then treated as read-only.
- **Insufficient Inputs:** Providing an array with fewer than two elements to `setInputs` will cause the node's inputs to become null. This will cause the `process` method to silently return 0.0, which can be extremely difficult to debug. Always ensure the input array has at least two valid Density objects.

## Data Pipeline
SmoothMaxDensity acts as a transformation step in the world generation data flow. It consumes two density values and outputs a single, blended value.

> Flow:
> Density.Context (3D Coordinates) -> Input A `process()` -> Value A
> <br>
> Density.Context (3D Coordinates) -> Input B `process()` -> Value B
> <br>
> (Value A, Value B) -> **SmoothMaxDensity** -> `Calculator.smoothMax()` -> Final Density `double`

