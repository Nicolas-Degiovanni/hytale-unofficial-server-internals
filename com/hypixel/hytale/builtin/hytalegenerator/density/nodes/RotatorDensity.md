---
description: Architectural reference for RotatorDensity
---

# RotatorDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class RotatorDensity extends Density {
```

## Architecture & Concepts
The RotatorDensity class is a fundamental transformation node within the procedural world generation framework. It functions as a spatial decorator in a directed acyclic graph (DAG) of Density nodes. Its primary role is not to generate density values itself, but to modify the coordinate system for its single child node, referred to as its *input*.

When the generator queries for a density value at a specific world position, the RotatorDensity intercepts this query. It applies a compound rotation to the position vector before passing the transformed coordinates to its input Density. This allows for the creation of complex, non-axis-aligned geological features, such as tilted pillars, angled ore veins, or swirling noise patterns, by rotating simpler, axis-aligned shapes.

The rotation is defined by two parameters: a target up-vector, `newYAxis`, and a `spinAngle`. The constructor pre-calculates the necessary rotational axes and angles to first tilt the standard world Y-axis to align with `newYAxis`, and then apply the specified spin around this new axis.

To ensure mathematical stability, the implementation includes special handling for cases where the `newYAxis` is parallel or anti-parallel to the world Y-axis, which would otherwise result in an undefined cross product. This is managed internally by the `SpecialCase` enum, making the node robust against common edge-case inputs.

## Lifecycle & Ownership
- **Creation:** RotatorDensity instances are not typically created manually. They are instantiated by a graph deserializer when it parses a world generation configuration file. The constructor requires an input Density and rotational parameters, establishing its position within the larger generation graph at creation time.
- **Scope:** The object's lifetime is bound to the world generator instance that contains it. It persists as a node in the generator's Density graph for the duration of a world generation session.
- **Destruction:** The instance is eligible for garbage collection when the parent world generator is discarded. This occurs, for example, when a server shuts down or a new world with a different generator configuration is loaded.

## Internal State & Concurrency
- **State:** This class is stateful. It caches the pre-calculated rotational parameters (`rotationAxis`, `tiltAxis`, `tiltAngle`, `spinAngle`) upon instantiation. These parameters are immutable for the lifetime of the object. The reference to the `input` Density, however, can be modified via the `setInputs` method.

- **Thread Safety:** The class is conditionally thread-safe. The `process` method is safe to call from multiple threads simultaneously because it operates on a *clone* of the incoming position vector, preventing side effects between concurrent evaluations.

    **WARNING:** The Density graph structure is not designed to be mutated while generation is in progress. Calling `setInputs` on a RotatorDensity instance while worker threads are actively calling its `process` method will lead to race conditions and undefined behavior. The entire Density graph should be considered immutable after the generation phase begins.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RotatorDensity(input, newYAxis, spinAngle) | constructor | O(1) | Creates a new rotator node. Pre-calculates all required rotational transforms. |
| process(context) | double | O(1) + O(input) | Clones the context's position, applies the rotation, and delegates to the input node. |
| setInputs(inputs) | void | O(1) | Sets the child Density node. Primarily used by the graph loader during initialization. |

## Integration Patterns

### Standard Usage
RotatorDensity is used declaratively within a generator configuration. It wraps another Density node to alter its orientation in the world. A developer would typically define this relationship in a configuration file, not in Java code.

```java
// Conceptual Usage during Graph Construction
// This code is typically executed by a configuration loader, not a game logic developer.

// Create the base shape we want to rotate
Density shapeToRotate = new PerlinNoiseDensity(...);

// Define the new "up" direction and a spin around that axis
Vector3d tiltDirection = new Vector3d(1.0, 1.0, 0.0).normalize();
double spin = 45.0;

// Wrap the shape in a RotatorDensity
Density rotatedShape = new RotatorDensity(shapeToRotate, tiltDirection, spin);

// The 'rotatedShape' node can now be used in the rest of the generation graph.
// When 'rotatedShape.process(context)' is called, it will internally call
// 'shapeToRotate.process()' with a rotated context.
```

### Anti-Patterns (Do NOT do this)
- **Graph Mutation During Generation:** Do not call `setInputs` on any node in a Density graph after the world generation process has started. This will break thread-safety guarantees.
- **Chaining Unnecessary Rotations:** Chaining multiple RotatorDensity nodes can lead to performance degradation and floating-point precision errors. Rotations should be combined into a single RotatorDensity where possible.
- **Ignoring Normalization:** Providing a `newYAxis` vector that is not normalized can lead to unexpected scaling behavior, as the internal calculations rely on unit vectors for pure rotation.

## Data Pipeline
The primary function of this class is to transform the spatial context for a subsequent node in the data pipeline.

> Flow:
> Density.Context In -> **RotatorDensity.process()** -> Clone & Rotate Vector3d -> New Density.Context Out -> Input.process() -> double Density Value

