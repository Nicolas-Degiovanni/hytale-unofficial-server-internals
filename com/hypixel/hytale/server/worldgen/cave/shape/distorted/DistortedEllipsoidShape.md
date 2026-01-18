---
description: Architectural reference for DistortedEllipsoidShape
---

# DistortedEllipsoidShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape.distorted
**Type:** Transient

## Definition
```java
// Signature
public class DistortedEllipsoidShape extends AbstractDistortedBody {
```

## Architecture & Concepts

The DistortedEllipsoidShape is a mathematical primitive that defines a volumetric ellipsoid used for procedural cave generation. It serves as a foundational building block for creating organic, non-uniform cave chambers and passages.

Architecturally, this class is a concrete implementation of a "distortable body" concept. It represents a perfect, mathematically-defined ellipsoid which is then warped in real-time by a separate noise function. This separation of concerns is critical: the DistortedEllipsoidShape defines the base geometry, while a passed-in ShapeDistortion object defines the surface irregularity and texture.

This component is not a service or manager; it is a short-lived data object that encapsulates the parameters of a single cave node's volume. Its primary role is to answer the question: "For a given (x, z) coordinate, what is the vertical height of the cave at this point?" The use of a GeneralNoise.InterpolationFunction allows the shape's boundary falloff to be configured, enabling smooth (quintic) or sharp (linear) transitions from air to solid rock.

### Lifecycle & Ownership
-   **Creation:** Instantiation is managed exclusively by the nested DistortedEllipsoidShape.Factory. This factory is invoked by higher-level world generation systems, such as a CaveNode processor, which provides the origin, orientation, and radius parameters. The factory contains critical logic for adjusting the Y and Z radii based on the pitch, ensuring the ellipsoid stretches and squashes realistically as the cave passage tilts up or down.
-   **Scope:** The object's lifetime is extremely brief and tied to a specific, localized generation task. It is created to define the volume of a single cave node, used by the world generator to sample block positions within its bounding box, and then becomes eligible for garbage collection.
-   **Destruction:** Cleanup is handled by the standard Java garbage collector. The class manages no external or native resources.

## Internal State & Concurrency
-   **State:** The object's state is **immutable**. All fields, including radii and pre-calculated squared values, are final and set only once within the constructor. This design ensures that a shape's definition cannot change after its creation.
-   **Thread Safety:** DistortedEllipsoidShape is inherently **thread-safe**. Its immutability guarantees that multiple world generation threads can safely call its methods concurrently without risk of data corruption or race conditions. This is a crucial feature for enabling high-performance, parallelized chunk generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHeight(seed, x, z, t, centerY, caveType, distortion) | double | O(1) | **Core Method.** Calculates the distorted vertical radius (height) of the ellipsoid at a given world (x, z) coordinate. Returns 0.0 if the coordinate is outside the shape's horizontal footprint. |
| getAnchor(vector, tx, ty, tz) | Vector3d | O(1) | Calculates a point on the surface of the underlying perfect sphere/ellipsoid. Used for connecting cave segments. |
| getWidthAt(t) | double | O(1) | Returns the minimum horizontal radius of the shape. |
| getHeightAt(t) | double | O(1) | Returns the vertical radius of the shape. |

## Integration Patterns

### Standard Usage
The intended use is through the factory pattern. A generator responsible for placing cave nodes will obtain an instance from the factory and use it to carve out a volume in the world.

```java
// Obtain the factory, typically from a registry
DistortedEllipsoidShape.Factory shapeFactory = getShapeFactory(CaveShape.ELLIPSOID);

// Create a new shape instance for a cave node
DistortedShape ellipsoid = shapeFactory.createShape(
    node.origin,
    node.direction,
    node.yaw,
    node.pitch,
    node.radiusX,
    node.radiusY,
    node.radiusZ,
    GeneralNoise.InterpolationFunction.QUINTIC
);

// During chunk generation, sample the shape's volume
// The 'distortion' object provides the noise function
double caveHeight = ellipsoid.getHeight(seed, blockX, blockZ, 0.0, node.origin.y, caveType, distortion);

if (blockY < node.origin.y + caveHeight && blockY > node.origin.y - caveHeight) {
    // This block is inside the cave
    chunk.setBlock(blockX, blockY, blockZ, Block.AIR);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new DistortedEllipsoidShape(...)`. This bypasses the critical radius and pitch adjustment logic inside the `Factory.createShape` method. Doing so will result in visually incorrect shapes for caves that are not perfectly horizontal.
-   **Reusing Instances:** Do not attempt to cache and reuse DistortedEllipsoidShape instances for different cave nodes. They are lightweight objects designed to be created and discarded, and their internal state is specific to a single origin and orientation.

## Data Pipeline
The DistortedEllipsoidShape is a processing stage within the broader cave generation data pipeline. It translates abstract cave node parameters into a concrete, queryable volume.

> Flow:
> Cave Node Parameters (Position, Radii, Orientation) -> **DistortedEllipsoidShape.Factory** -> **DistortedEllipsoidShape Instance** -> World Generator Sampler -> Block State Modification in Chunk Buffer

