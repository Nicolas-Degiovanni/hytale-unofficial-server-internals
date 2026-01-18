---
description: Architectural reference for AxisDensity
---

# AxisDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient / Data Model

## Definition
```java
// Signature
public class AxisDensity extends Density {
```

## Architecture & Concepts
AxisDensity is a fundamental node within the procedural world generation system. It implements the `Density` contract, designed to produce a density field that is radially symmetric around a defined 3D axis. In conceptual terms, it generates shapes like infinite cylinders, tubes, or beams of density through the world space.

The behavior of this node is controlled by three key parameters:
1.  **axis:** A Vector3d that defines the direction and orientation of the central line. The magnitude of this vector is not used.
2.  **distanceCurve:** A function that maps the perpendicular distance from a sample point to the axis into a final density value. This curve dictates the profile of the shape, such as creating a sharp-edged cylinder, a soft-edged tube, or more complex concentric rings.
3.  **isAnchored:** A boolean flag that alters the coordinate system.
    *   When **false**, the axis is assumed to pass through the world origin (0, 0, 0).
    *   When **true**, the axis is relative to a `densityAnchor` position provided by the `Density.Context`. This allows the same shape to be translated and stamped at various locations without re-creating the node.

This class is a core component for creating large-scale linear features in the world, such as massive tree trunks, cave tunnels, or abstract geometric patterns.

## Lifecycle & Ownership
- **Creation:** An AxisDensity instance is created and configured by a higher-level component, such as a biome generator or a feature placement system. Its entire configuration (axis, curve, anchor mode) is provided at construction time.
- **Scope:** The object's lifetime is typically bound to the world generation session or the specific generator graph it is a part of. As it is immutable, a single instance can be safely reused for millions of density calculations across the world.
- **Destruction:** The instance is marked for garbage collection when the generator that created it is discarded. It holds no external resources that require manual cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. The internal fields `distanceCurve`, `axis`, and `isAnchored` are final and set exclusively by the constructor. The object's state cannot be modified after instantiation.
- **Thread Safety:** **Fully Thread-Safe**. Due to its immutable design, an AxisDensity object can be safely accessed by multiple world-generation worker threads simultaneously without any need for external locking or synchronization. The `process` method is a pure function of its inputs and immutable state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) | Calculates the density for the position specified in the context. Returns 0.0 if the axis has no length or if an anchor is required but not provided. |

## Integration Patterns

### Standard Usage
The standard pattern involves configuring an AxisDensity node once and then reusing it repeatedly during the density evaluation phase of world generation.

```java
// 1. Define a curve function that creates a solid tube with a radius of 8 units.
Double2DoubleFunction tubeCurve = distance -> (distance < 8.0) ? 1.0 : 0.0;

// 2. Define an axis, for example, a vertical one.
Vector3d verticalAxis = new Vector3d(0, 1, 0);

// 3. Instantiate the node for a non-anchored, world-origin cylinder.
AxisDensity verticalTubeGenerator = new AxisDensity(tubeCurve, verticalAxis, false);

// 4. During world generation, evaluate a point's density.
Density.Context evalContext = new Density.Context(new Vector3d(4, 50, 0));
double density = verticalTubeGenerator.process(evalContext);
// In this case, 'density' would be 1.0 because the point (4, 50, 0) is 4 units
// away from the vertical axis, which is less than the 8.0 unit radius.
```

### Anti-Patterns (Do NOT do this)
- **Missing Anchor:** Do not call `process` on a node where `isAnchored` is true without first ensuring the `Density.Context` contains a non-null `densityAnchor`. Failure to do so will cause the method to return 0.0, potentially creating large, unintended empty volumes in the generated world.
- **Zero-Length Axis:** Avoid constructing an AxisDensity with a zero-vector for the `axis`. The class contains a safety check that returns 0.0, but this indicates a configuration error and results in a computationally useless node.
- **Mutable Vector State:** Do not modify the internal state of the `axis` Vector3d object after passing it to the constructor. The system's thread safety and determinism rely on this vector remaining constant throughout the object's lifetime.

## Data Pipeline
AxisDensity acts as a transformation node within a larger data processing graph. It does not manage a pipeline itself but is a critical step within one.

> Flow:
> World Generator -> Provides `Density.Context` (with position and optional anchor) -> **AxisDensity.process** -> Calculates distance to line -> Transforms distance via `distanceCurve` -> Returns `double` density value -> Aggregator Node

