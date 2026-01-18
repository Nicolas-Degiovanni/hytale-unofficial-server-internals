---
description: Architectural reference for Distance2AddReturnType
---

# Distance2AddReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Strategy Object

## Definition
```java
// Signature
public class Distance2AddReturnType extends ReturnType {
```

## Architecture & Concepts
The Distance2AddReturnType is a specific strategy node within the Hytale World Generator's density function framework. It is not a standalone service but a component in a larger computational graph that defines terrain and biome shapes.

Its primary function is to implement a specific mathematical formula for combining two distance values into a single, normalized density value. This class embodies the *additive distance* combination strategy. The core formula, `(distance0 + distance1) / maxDistance - 1.0`, is used to create smooth transitions or blended regions between two geometric features defined elsewhere in the density graph. The result is typically a value between -1.0 and 1.0, which the world generator interprets to determine if a point in space is solid, air, or something in between.

This class is a leaf node in its operational branch; it performs a final calculation based on inputs provided by parent nodes and returns a primitive double, which is then propagated back up the evaluation chain.

### Lifecycle & Ownership
- **Creation:** Instances are created by the world generation asset loader when parsing a density function graph from a configuration file. It is not intended for manual instantiation during gameplay.
- **Scope:** The object's lifetime is bound to the lifecycle of the world generator's currently loaded density graph. It persists as long as that specific world generation configuration is active.
- **Destruction:** The object is eligible for garbage collection when the world generator discards its density graph, for instance, when a world is unloaded or a different generator configuration is loaded.

## Internal State & Concurrency
- **State:** This class is effectively immutable after its initial configuration. Its behavior is controlled by the inherited `maxDistance` field, which is set by the asset loader and does not change during the object's lifetime. The `get` method is a pure function with no side effects.
- **Thread Safety:** This class is unconditionally thread-safe. The `get` method operates solely on its arguments and the immutable `maxDistance` field. It can be invoked concurrently by multiple world generation worker threads without any need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(distance0, distance1, samplePosition, closestPoint0, closestPoint1, context) | double | O(1) | Computes a density value by summing two distances and normalizing by maxDistance. This is the core operation of the class. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation by game logic or external systems. It is used exclusively by the density function evaluation engine. The engine identifies this object as the return type strategy for a parent node and calls its `get` method to finalize a density calculation for a specific point in space.

```java
// Conceptual example of how the world generator engine would use this class.
// A developer would NOT write this code.

// Engine retrieves the strategy from a pre-configured parent node
ReturnType strategy = parentDensityNode.getReturnTypeStrategy(); // This is a Distance2AddReturnType instance

// Engine calculates prerequisite values
double d0 = ...;
double d1 = ...;
Vector3d c0 = ...;
Vector3d c1 = ...;

// Engine invokes the strategy
double finalDensity = strategy.get(d0, d1, samplePos, c0, c1, densityContext);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new Distance2AddReturnType()`. The behavior of this class is dependent on the `maxDistance` field, which must be configured by the asset loading pipeline from world generation files. Manual creation will result in an unconfigured and non-functional object.
- **State Mutation:** Do not attempt to modify the `maxDistance` field via reflection after initialization. The world generator assumes this value is constant, and changing it at runtime will lead to non-deterministic and visually corrupt world generation.

## Data Pipeline
This class acts as a terminal calculation step within a larger data flow for determining the density at a single point in 3D space.

> Flow:
> World Generator requests density at **Vector3d** -> Density Graph Evaluator traverses nodes -> Parent Node calculates **distance0** and **distance1** -> Parent Node invokes **Distance2AddReturnType.get()** -> Returned **double** is propagated up the graph for further processing.

