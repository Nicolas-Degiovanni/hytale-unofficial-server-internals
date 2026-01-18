---
description: Architectural reference for AbstractDistortedBody
---

# AbstractDistortedBody

**Package:** com.hypixel.hytale.server.worldgen.cave.shape.distorted
**Type:** Transient

## Definition
```java
// Signature
public abstract class AbstractDistortedBody extends AbstractDistortedShape {
```

## Architecture & Concepts

AbstractDistortedBody is a foundational abstract class within the server-side procedural cave generation framework. It serves as the mathematical backbone for any cave segment that can be represented as a volumetric shape with a specific origin, orientation, and set of radii.

Its primary architectural role is to decouple the complex, world-space coordinate geometry from the simple, local-space shape definition. It achieves this by managing a `CoordinateRotator` instance, which transforms incoming world-space query coordinates (x, z) into the local coordinate system of the shape. This allows concrete subclasses, such as a cylinder or ellipsoid, to define their geometry in a simple, un-rotated space, dramatically reducing their implementation complexity.

This class embodies the Template Method design pattern. The public method `getHeightAtProjection` defines a fixed algorithm for coordinate transformation, but defers the final height calculation to the abstract `getHeight` method, which must be implemented by subclasses. This enforces a consistent transformation pipeline for all cave shapes derived from it.

## Lifecycle & Ownership

-   **Creation:** Instances are not created directly. They are instantiated by a concrete subclass's nested `Factory` implementation (e.g., `CylinderDistortedBody.Factory`). This factory is invoked by a higher-level cave generation orchestrator when a specific cave segment needs to be realized.
-   **Scope:** The object's lifetime is ephemeral and tied to the generation of a single cave segment. It holds the geometric definition for that segment while the world generator queries it to place blocks.
-   **Destruction:** Once the relevant world chunk or region has been generated, the AbstractDistortedBody instance is no longer referenced and becomes eligible for standard garbage collection. It manages no persistent resources that require explicit cleanup.

## Internal State & Concurrency

-   **State:** The internal state, consisting of the origin vector `o`, direction vector `v`, and the `rotation` matrix, is established at construction and is effectively immutable. The fields are declared `final`, ensuring that the shape's position and orientation cannot be altered post-instantiation.
-   **Thread Safety:** This class is inherently thread-safe. Its state is immutable, and its methods are pure functions that perform calculations without causing side effects. A single instance can be safely read by multiple world generation threads simultaneously without the need for external locking or synchronization.

## API Surface

The public contract is designed for querying the shape's geometry from world space.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHeightAtProjection(...) | double | O(1) | **Primary entry point.** Transforms world coordinates into the shape's local space and returns the vertical thickness (height) at that point by calling the abstract `getHeight` method. |
| getFloor(...) | double | O(1) | Calculates the absolute world Y-coordinate of the cave floor for a given point, accounting for the shape's rotation. |
| getCeiling(...) | double | O(1) | Calculates the absolute world Y-coordinate of the cave ceiling for a given point, accounting for the shape's rotation. |
| getHeight(...) | abstract double | Varies | **Subclass responsibility.** Defines the core shape logic. Calculates the vertical thickness of the cave in local coordinates. |

## Integration Patterns

### Standard Usage

The class is intended to be extended. A higher-level system uses the associated factory to create a concrete instance and then queries it to define a volume in the world.

```java
// A hypothetical generator using a concrete implementation's factory
DistortedShape.Factory factory = new ConcreteDistortedBody.Factory();

// Factory creates the shape from high-level parameters
DistortedShape caveSegment = factory.create(origin, direction, length, ...);

// Generator queries the shape to determine cave boundaries
for (int x = startX; x < endX; x++) {
    for (int z = startZ; z < endZ; z++) {
        double height = caveSegment.getHeightAtProjection(seed, x, z, ...);
        if (height > 0) {
            double floorY = caveSegment.getFloor(x, z, centerY, height);
            double ceilY = caveSegment.getCeiling(x, z, centerY, height);
            // ... logic to place blocks between floorY and ceilY
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not attempt to construct this class or its subclasses directly using `new`. The nested `Factory` is the designated creation mechanism, as it correctly calculates yaw and pitch from a direction vector, which is a non-trivial and error-prone operation.
-   **State Mutation:** Do not attempt to modify the internal vectors or `CoordinateRotator` after construction. The object's immutability is critical for predictable and thread-safe generation.

## Data Pipeline

This class sits in the middle of the cave generation pipeline, translating abstract definitions into concrete geometric queries.

> Flow:
> Cave System Planner -> Provides (Origin, Direction, Dimensions) -> **AbstractDistortedBody.Factory** -> Instantiates Concrete Shape -> World Generator queries with (X, Z) coordinates -> **AbstractDistortedBody** transforms coordinates and returns height -> Voxel Placement Logic

