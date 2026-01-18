---
description: Architectural reference for ManhattanDistanceFunction
---

# ManhattanDistanceFunction

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.distancefunctions
**Type:** Transient

## Definition
```java
// Signature
public class ManhattanDistanceFunction extends DistanceFunction {
```

## Architecture & Concepts
The ManhattanDistanceFunction is a stateless, concrete implementation of the abstract DistanceFunction. It serves as a swappable component within Hytale's procedural world generation framework, specifically within the density graph system.

Its primary role is to calculate the L1 norm, or "taxicab geometry" distance, of a 3D vector. This metric is defined as the sum of the absolute differences of the coordinates. In the context of world generation, this function is used to create distance fields that exhibit sharp, grid-aligned falloffs, resulting in square or diamond-shaped patterns. This is in stark contrast to the circular or spherical patterns produced by a standard EuclideanDistanceFunction.

This class embodies the **Strategy Pattern**, allowing higher-level generator nodes to be configured with different distance calculation algorithms without changing their core logic. It is a fundamental building block for creating artificial or stylized terrain features where natural, rounded shapes are undesirable.

### Lifecycle & Ownership
- **Creation:** Instances are typically created by a deserializer when loading a world generation graph from configuration files. A generator node that requires a distance calculation will be configured to instantiate and hold a reference to this specific function.
- **Scope:** The object is transient and its lifetime is tied to its owning generator node. It may be re-created frequently as different parts of the generation graph are evaluated.
- **Destruction:** The object is managed by the Java garbage collector. As it is stateless and holds no external resources, its destruction is a low-overhead operation once it is no longer referenced by a generator node.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no member variables and its output depends solely on its input arguments. Each call to getDistance is an independent, pure function execution.
- **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless nature, a single instance can be safely shared and called by multiple world generation threads simultaneously without any risk of race conditions or need for synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDistance(Vector3d point) | double | O(1) | Calculates the Manhattan distance from the origin (0,0,0) to the specified point. The input vector must not be null. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation in general game logic. It is designed to be injected into a more complex system, such as a distance field or density noise generator, which then calls getDistance internally for each point in a volume.

```java
// A hypothetical generator node configured to use this function
// Note: This is a conceptual example.
DistanceFieldNode fieldNode = new DistanceFieldNode();
DistanceFunction manhattan = new ManhattanDistanceFunction();

// The node is configured with the specific distance strategy
fieldNode.setDistanceFunction(manhattan);

// The node internally calls getDistance for many points
double density = fieldNode.computeDensityAt(new Vector3d(10, -20, 5));
```

### Anti-Patterns (Do NOT do this)
- **Misuse for Natural Falloff:** Do not use this function when a natural, circular, or spherical distance field is required. Its grid-aligned calculation will produce visually jarring square-like shapes that are unsuitable for organic terrain features like hills or valleys.
- **Redundant Instantiation:** While safe, creating a new ManhattanDistanceFunction for every single distance calculation is inefficient. An owning generator node should instantiate it once and reuse that instance for all its computations.

## Data Pipeline
The function acts as a simple transformation step within a larger world generation data pipeline. It consumes a coordinate and produces a scalar value representing distance.

> Flow:
> Generator Node (requests point evaluation) -> **ManhattanDistanceFunction.getDistance(point)** -> double (raw distance value) -> Subsequent Processing Node (e.g., a curve mapper or noise combiner)

