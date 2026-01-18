---
description: Architectural reference for Distance2DivReturnType
---

# Distance2DivReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Transient

## Definition
```java
// Signature
public class Distance2DivReturnType extends ReturnType {
```

## Architecture & Concepts
The Distance2DivReturnType is a specialized calculation strategy used within Hytale's procedural world generation framework. It operates as a component within a larger density function graph, which collectively determines the substance and shape of the game world at a granular level.

This class implements a specific mathematical transformation that takes two distance values as input and computes a single, normalized output. Its core purpose is to model the *relative influence* of two spatial features. The calculation, `(distance0 / distance1) * 2.0 - 1.0`, creates a gradient field based on the ratio of a sample point's proximity to two distinct locations. This is commonly used to define smooth transitions, boundaries, or regions of influence between geological features, biomes, or other generated structures.

It is designed to be a pluggable component, allowing world designers to select this return type within a density graph node to achieve this specific mathematical effect without altering the graph's structure.

### Lifecycle & Ownership
- **Creation:** Instances are not created dynamically during gameplay. They are instantiated once during the world generator's initialization phase, typically by a configuration service that parses world generation asset files (e.g., JSON definitions).
- **Scope:** The object's lifetime is bound to the world generator configuration it is part of. It persists as long as that generator is active for a given world.
- **Destruction:** The object is marked for garbage collection when the parent world generator is unloaded, for instance, when a server shuts down or a different world is loaded.

## Internal State & Concurrency
- **State:** This class is effectively immutable after configuration. It inherits the `maxDistance` field from its parent, which is set upon creation and not modified thereafter. The `get` method does not alter any internal state.
- **Thread Safety:** The Distance2DivReturnType is unconditionally thread-safe. Its `get` method is a pure function that depends only on its input arguments and the immutable `maxDistance` field. Multiple world generation worker threads can safely invoke `get` on a shared instance without synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(distance0, distance1, ...) | double | O(1) | Computes a density value from the ratio of two distances. Returns 1.0 if the primary closest point is null. Returns 0.0 if the configured maxDistance is non-positive. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by general game systems. It is used exclusively by parent density nodes within the world generator. A parent node, responsible for finding closest feature points, would delegate the final value calculation to an instance of this class.

```java
// Conceptual example within a parent Density Node
// The 'returnType' field is configured to be a Distance2DivReturnType instance

public double computeDensity(Vector3d samplePosition) {
    // ... logic to find two closest points and their distances ...
    double d0 = findDistanceToPointA(samplePosition);
    double d1 = findDistanceToPointB(samplePosition);
    
    // Delegate final calculation to the configured strategy
    return this.returnType.get(d0, d1, samplePosition, ...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new Distance2DivReturnType()` in application code. These objects must only be created via the world generation asset pipeline to ensure they are configured correctly.
- **State Mutation:** Although the `maxDistance` field is accessible, it must not be modified after initialization. Doing so would violate thread safety and lead to non-deterministic world generation.
- **Ignoring Edge Cases:** The formula can result in division by zero if `distance1` is zero, producing an `Infinity` result. The density graph must be designed to handle or clamp such values.

## Data Pipeline
The Distance2DivReturnType acts as a transformation step in the world generation data pipeline. It consumes distance metrics and produces a single scalar value that contributes to the final density of a voxel.

> Flow:
> World Generator Request -> Density Graph Evaluation -> Parent Node (calculates distances) -> **Distance2DivReturnType.get()** -> Normalized Density Value -> Aggregator Node -> Final Voxel Material ID

