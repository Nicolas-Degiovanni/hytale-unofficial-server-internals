---
description: Architectural reference for AbstractDistortedExtrusion
---

# AbstractDistortedExtrusion

**Package:** com.hypixel.hytale.server.worldgen.cave.shape.distorted
**Type:** Template Class

## Definition
```java
// Signature
public abstract class AbstractDistortedExtrusion extends AbstractDistortedShape {
```

## Architecture & Concepts
AbstractDistortedExtrusion is a foundational component in the procedural cave generation system. It provides a generalized algorithm for creating volumetric cave segments by extruding a 2D cross-section along a 3D path. The path and its basic dimensions are managed by its parent, AbstractDistortedShape.

The core architectural principle of this class is the **Template Method Pattern**. It defines a fixed, multi-stage algorithm in the `getHeightAtProjection` method for calculating a cave's ceiling height, but defers the specific geometric definition of the cross-section to concrete subclasses.

This class orchestrates the interaction between several world generation concepts:
1.  **Base Shape:** The fundamental path and dimensions (width, height) inherited from AbstractDistortedShape.
2.  **Cave Type:** A CaveType instance provides high-level modifiers, such as a `heightRadiusFactor`, allowing different cave biomes to influence the shape's scale.
3.  **Distortion:** A ShapeDistortion object injects noise and organic variation into the shape's width, preventing geometrically perfect and unnatural-looking caves.
4.  **Interpolation:** A GeneralNoise.InterpolationFunction is used to define the "falloff" curve of the cave's ceiling, controlling whether the cross-section has sharp edges, smooth curves, or other profiles.

Subclasses are only responsible for implementing the raw geometry via `getDistanceSq` and `getHeightComponent`, completely decoupling the cross-section shape from the complex distortion and blending logic.

## Lifecycle & Ownership
-   **Creation:** Instances are created by high-level generation controllers, such as a CaveCarver or a biome-specific factory, when a new cave segment is required. A concrete implementation (e.g., a cylindrical or box-shaped extrusion) is always instantiated, not this abstract base.
-   **Scope:** The object is transient and its lifetime is strictly bound to the generation of a single cave feature. It holds no long-term state and is discarded after its geometry has been evaluated.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the world generation algorithm finishes querying it for height values and releases all references.

## Internal State & Concurrency
-   **State:** This class is **effectively immutable**. Its only field, `interpolation`, is final and set during construction. All other state is inherited from its parent and is also established at creation time. The class performs calculations but does not cache results or modify its internal state after instantiation.
-   **Thread Safety:** **Thread-safe**. Due to its immutable design, a single instance can be safely read and executed by multiple world generation worker threads simultaneously without any need for external locking or synchronization. The `getHeightAtProjection` method is a pure function of its inputs and the object's construction-time configuration.

## API Surface
The public contract is focused on the final height calculation, while the protected contract defines the geometric primitives required from subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHeightAtProjection(seed, x, z, t, centerY, caveType, distortion) | double | O(1) | **Primary Entry Point.** Calculates the final, distorted cave ceiling height at a given world coordinate. This method orchestrates all modifiers and interpolation. |
| getDistanceSq(var1, var3, var5) | protected abstract double | Varies | **Subclass Contract.** Must be implemented to return the squared distance from a point to the shape's central axis. Critical for performance as it avoids square roots. |
| getHeightComponent(var1, var3, var5) | protected abstract double | Varies | **Subclass Contract.** Must be implemented to return the raw, pre-interpolation height contribution (alpha) based on the shape's geometry. |

## Integration Patterns

### Standard Usage
A concrete subclass is instantiated by a factory or manager responsible for a specific type of cave. The world generator then iterates over a grid of world coordinates, calling `getHeightAtProjection` for each point to determine the cave's volume.

```java
// Example of a hypothetical subclass and its usage
// NOTE: CylindricalDistortedExtrusion is a hypothetical concrete class
AbstractDistortedExtrusion caveSegment = new CylindricalDistortedExtrusion(origin, direction, 10.0, 5.0, GeneralNoise.InterpolationFunction.SMOOTHSTEP);

// The world generator would then call this repeatedly
double ceilingHeight = caveSegment.getHeightAtProjection(seed, x, z, t, y, caveType, distortion);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Subclasses:** Avoid creating subclasses that contain mutable state. The world generation engine relies on the thread-safety of these shapes. Introducing mutable fields without proper synchronization will cause severe and difficult-to-debug race conditions and generation artifacts.
-   **Expensive Geometry Calculations:** The methods `getDistanceSq` and `getHeightComponent` will be called millions of times during world generation. Implementations must be highly performant. Avoid costly operations like square roots (favor squared-distance comparisons), complex iterations, or memory allocations.

## Data Pipeline
This component acts as a computational node in the world generation pipeline. It transforms a set of input parameters into a single height value.

> Flow:
> (World Coordinates, Seed, CaveType, Distortion) -> **getHeightAtProjection** -> [Subclass `getDistanceSq`] -> [Subclass `getHeightComponent`] -> [Interpolation Function] -> Final Height (double)

