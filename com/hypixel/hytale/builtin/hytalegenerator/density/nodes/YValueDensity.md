---
description: Architectural reference for YValueDensity
---

# YValueDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Utility

## Definition
```java
// Signature
public class YValueDensity extends Density {
```

## Architecture & Concepts
The YValueDensity class is a foundational component within Hytale's procedural world generation framework. It functions as a *source node* or *leaf node* in a Density Function graph. Its sole responsibility is to provide the raw vertical (Y-axis) coordinate of the point currently being evaluated by the world generator.

Architecturally, it serves as one of the primary inputs for creating height-dependent environmental features. By injecting the world-space Y-coordinate into the density graph, designers can construct complex rules for material distribution, biome layering, and structural placement. For example, it can be used to define a water table, transition between rock strata at different depths, or control the density of air blocks above a certain altitude.

The implementation of an empty `setInputs` method is a deliberate design choice, signaling that this node does not depend on the output of other density nodes. It is a terminal provider of a single, primitive value, making it a fundamental building block for more complex density calculations.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly in code. They are instantiated by the world generator's configuration loader when parsing a declarative asset file, such as a JSON definition of a density graph.
- **Scope:** The object's lifetime is bound to the world generator configuration it is a part of. It persists in memory as long as that specific generator is active and in use.
- **Destruction:** The instance is marked for garbage collection when the parent world generator configuration is unloaded or replaced.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its output is purely a function of its input arguments. It does not cache any data.
- **Thread Safety:** YValueDensity is inherently **thread-safe**. Due to its stateless nature, a single shared instance can be safely processed by multiple world generation worker threads concurrently without locks or synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) | Returns the Y-coordinate from the provided context. Throws NullPointerException if the context or its position is null. |
| setInputs(Density[] inputs) | void | O(1) | No-op. This node does not accept inputs. Calling this method has no effect. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or invocation by game systems. It is designed to be a component within a larger, data-driven density graph. The world generator engine is responsible for constructing the graph and invoking the `process` method during terrain evaluation.

The following conceptual example illustrates how the *engine* might use a YValueDensity node as part of a larger calculation to create a simple water plane.

```java
// Conceptual Engine Code - Do NOT replicate
// Assume 'yValueNode' is a YValueDensity instance loaded from a config.
// Assume 'waterLevelNode' is a ConstantDensity(64) instance.

// This node subtracts the water level from the current Y position.
// The result is negative underwater, positive above water.
SubtractDensity subtractNode = new SubtractDensity();
subtractNode.setInputs(new Density[]{ yValueNode, waterLevelNode });

// The engine evaluates the graph for a specific world coordinate.
Density.Context context = new Density.Context(new Vector3d(100, 50, 200));
double result = subtractNode.process(context); // result will be -14
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new YValueDensity()` in gameplay or system logic. World generation functions should be defined entirely within declarative asset files to allow for hot-reloading and designer iteration.
- **Providing Inputs:** Calling `setInputs` on a YValueDensity instance is meaningless and indicates a misunderstanding of its role as a source node. The provided inputs will be ignored.

## Data Pipeline
YValueDensity acts as an entry point for coordinate data into the density graph. It transforms a positional context into a scalar value that flows to downstream nodes.

> Flow:
> World Generator Voxel Request -> Density.Context Creation (x,y,z) -> **YValueDensity.process(context)** -> double (y) -> Parent Density Node (e.g., Add, Subtract, Clamp)

