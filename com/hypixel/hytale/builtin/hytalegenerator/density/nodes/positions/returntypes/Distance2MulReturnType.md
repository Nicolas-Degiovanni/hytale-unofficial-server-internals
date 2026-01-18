---
description: Architectural reference for Distance2MulReturnType
---

# Distance2MulReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Transient

## Definition
```java
// Signature
public class Distance2MulReturnType extends ReturnType {
```

## Architecture & Concepts
The Distance2MulReturnType is a stateless strategy component within the procedural world generation's density graph system. It implements a specific algorithm for combining two distance values into a single, normalized density value.

Its primary role is to serve as a "blending function" in a declarative node-based system. In procedural generation, complex shapes are often formed by combining simpler geometric primitives. This class provides the logic for a multiplicative blend, a technique common in Signed Distance Field (SDF) modeling. The formula it employs, `(d1/max) * (d0/max) * 2.0 - 1.0`, normalizes the input distances, multiplies them, and maps the result to the standard density range of -1.0 to 1.0. This specific operation can be used to create smooth intersections or other complex composite shapes from two separate distance fields.

This class is not intended for direct use but is instead instantiated by the world generator's configuration loader based on definitions in world generation data files.

### Lifecycle & Ownership
- **Creation:** Instantiated by the density graph deserializer when parsing world generation configuration files. It is never created manually by game logic code.
- **Scope:** The object's lifetime is bound to the loaded world generator configuration. It persists as a node within the in-memory density graph for the duration of the server's or client's session.
- **Destruction:** Marked for garbage collection when the world generator configuration is unloaded, such as on server shutdown or when transitioning to a different world.

## Internal State & Concurrency
- **State:** Effectively immutable. The class inherits the `maxDistance` field from its parent, which is configured at creation and not modified thereafter. It contains no other mutable state.
- **Thread Safety:** This class is inherently thread-safe. The `get` method is a pure function with no side effects. Its behavior depends only on its arguments and its immutable configuration. It can be invoked concurrently by any number of world generation threads without requiring synchronization, which is critical for scalable and performant terrain generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(distance0, distance1, samplePosition, closestPoint0, closestPoint1, context) | double | O(1) | Computes a density value by normalizing and multiplying two input distances. Returns a value in the approximate range of -1.0 to 1.0. Returns a default value of 1.0 if the primary closest point is null. |

## Integration Patterns

### Standard Usage
This class is not invoked directly. It is a component of a larger density graph that is evaluated by the world generation engine. The engine traverses the graph, calling the `get` method as part of the evaluation pipeline.

```java
// Conceptual example of how the engine might use a ReturnType instance
// Note: This is a simplified representation. Developers do not write this code.

// Engine retrieves the configured return type node from the graph
ReturnType returnType = densityGraph.getReturnTypeNode("myDistanceMultiplier");

// Engine calculates distances from other nodes
double dist0 = positionNodeA.getDistance(samplePos);
double dist1 = positionNodeB.getDistance(samplePos);
Vector3d cp0 = positionNodeA.getClosestPoint(samplePos);
Vector3d cp1 = positionNodeB.getClosestPoint(samplePos);

// Engine invokes the get method to combine the results
double finalDensity = returnType.get(dist0, dist1, samplePos, cp0, cp1, context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never construct this class with `new Distance2MulReturnType()`. The world generator's configuration system is solely responsible for its creation and lifecycle. Manual instantiation bypasses the configuration system and will result in an unconfigured, non-functional object.
- **State Mutation:** Do not attempt to modify the inherited `maxDistance` field after initialization. The immutability of density graph nodes is a core assumption for ensuring consistent and thread-safe world generation.

## Data Pipeline
The class operates as a terminal processing step for distance-calculating nodes within the density graph.

> Flow:
> World Coordinate -> Position Node A -> (distance0, closestPoint0) -> **Distance2MulReturnType** -> Final Density Value
> World Coordinate -> Position Node B -> (distance1, closestPoint1) -> **Distance2MulReturnType** -> Final Density Value

