---
description: Architectural reference for Distance2ReturnType
---

# Distance2ReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Transient Component

## Definition
```java
// Signature
public class Distance2ReturnType extends ReturnType {
```

## Architecture & Concepts
The Distance2ReturnType is a specialized component within Hytale's procedural world generation framework. It functions as a **strategy object** within the density graph evaluation pipeline, responsible for translating raw distance measurements into a normalized density value.

In procedural generation, density functions determine the shape of the world, defining whether a given point in space is solid, air, or somewhere in between. This class implements a specific algorithm for this translation, commonly used in systems based on Signed Distance Fields (SDFs). Its primary role is to take the distance to a *secondary* geometric feature (distance1) and map it to the standard density range of [-1.0, 1.0], where negative values typically represent "inside" or solid material, and positive values represent "outside" or air.

This component is not used in isolation. It is configured and invoked by a parent density node, which is responsible for calculating the initial distance metrics that serve as this class's input. The specific logic handles edge cases, such as when a sample point is outside the influence of a primary shape, ensuring predictable and stable output from the density graph.

## Lifecycle & Ownership
- **Creation:** Instances of Distance2ReturnType are not created directly by developers. They are instantiated by the world generator's configuration loader, typically a parser that reads and constructs a density graph from a definition file (e.g., JSON). The parser creates this object when it encounters the corresponding node type.
- **Scope:** The object's lifetime is bound to the parent density graph configuration. It is a stateless, reusable component that persists as long as the world generation preset is active.
- **Destruction:** The object is marked for garbage collection when the density graph it belongs to is unloaded. This occurs when the server changes world generation presets or shuts down.

## Internal State & Concurrency
- **State:** This class is effectively immutable after its initial configuration. Its behavior is dictated by the `maxDistance` field inherited from the `ReturnType` parent class. This value is set once during the density graph's construction and is not modified during runtime evaluation.
- **Thread Safety:** The `get` method is pure, re-entrant, and unconditionally thread-safe. It performs calculations solely on its input arguments and its immutable `maxDistance` field. This design is critical for the performance of the world generator, which heavily parallelizes chunk generation across multiple threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(distance0, distance1, samplePosition, closestPoint0, closestPoint1, context) | double | O(1) | Calculates a normalized density value. It transforms `distance1` into the [-1.0, 1.0] range based on the configured `maxDistance`. Returns 1.0 if `closestPoint0` is null, indicating the sample is outside the primary shape's influence. |

## Integration Patterns

### Standard Usage
This class is an internal implementation detail of the density graph system and should not be invoked directly. A parent node, responsible for distance calculations, holds a reference to a `ReturnType` and calls its `get` method during evaluation.

```java
// Hypothetical usage within a parent Density Node
// NOTE: This is a conceptual example. Do not use this class directly.

// During density evaluation for a given samplePosition...
ReturnType returnTypeStrategy = this.configuredReturnType; // An instance of Distance2ReturnType

// Raw distances are calculated by the parent node
double d0 = calculateDistanceToShape0(samplePosition);
double d1 = calculateDistanceToShape1(samplePosition);
Vector3d p0 = findClosestPointOnShape0(samplePosition);
Vector3d p1 = findClosestPointOnShape1(samplePosition);

// The strategy is invoked to get the final density value
double density = returnTypeStrategy.get(d0, d1, samplePosition, p0, p1, context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new Distance2ReturnType()`. The object is useless without the `maxDistance` field being properly configured by the world generation framework. Direct creation will lead to a default `maxDistance` of 0.0, causing the `get` method to always return 0.0 and resulting in broken terrain.
- **State Modification:** Do not attempt to modify the `maxDistance` field after initialization. The world generator assumes this value is constant, and runtime changes will lead to non-deterministic behavior and visible seams between world chunks.

## Data Pipeline
The flow of data through this component is linear and unidirectional, acting as a transformation step within a larger evaluation chain.

> Flow:
> World Generator requests density at **Vector3d** -> Density Graph Traversal -> Parent Node calculates raw distances -> **Distance2ReturnType.get()** -> Normalized density **double** [-1, 1] -> Value is combined with other nodes -> Final density for position

