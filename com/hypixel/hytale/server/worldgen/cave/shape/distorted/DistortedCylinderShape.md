---
description: Architectural reference for DistortedCylinderShape
---

# DistortedCylinderShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape.distorted
**Type:** Transient Model

## Definition
```java
// Signature
public class DistortedCylinderShape extends AbstractDistortedExtrusion {
```

## Architecture & Concepts
The DistortedCylinderShape is a fundamental geometric primitive within the server-side procedural world generation engine. It serves as a mathematical model for a single segment of a cave tunnel, connecting two nodes in a cave network graph.

Architecturally, it is designed as a pure, stateless calculator. It does not hold game state or interact with other engine systems. Its sole responsibility is to define a volumetric shape in 3D space and provide an API to query its properties, such as its boundary, cross-sectional dimensions, and distance from its central axis.

The "distorted" aspect is key to its design. Rather than representing a perfect cylinder, it defines a shape whose width and height vary along its length. This variation is controlled by interpolating between start, middle, and end dimensions, allowing for the generation of more organic and natural-looking cave passages.

A critical, non-obvious feature is the automatic **pitch compensation**. The internal factory algorithmically scales the shape's height based on the steepness of its direction vector. This is essential for visual fidelity, preventing tunnels on steep inclines from appearing unnaturally squashed or flattened.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the nested DistortedCylinderShape.Factory. This factory is not meant for direct developer use but is invoked by higher-level procedural generation orchestrators, such as a `CaveNetworkGenerator`, when constructing a tunnel between two points. The factory pattern is mandatory here as it encapsulates the critical pitch compensation logic.

- **Scope:** Extremely short-lived and function-scoped. An instance of DistortedCylinderShape typically exists only for the duration of a single chunk's voxelization process. It is created, used to test thousands of block positions, and then immediately becomes eligible for garbage collection.

- **Destruction:** Automatic. The Java Garbage Collector reclaims the memory. No manual cleanup or resource release is necessary due to its purely mathematical nature.

## Internal State & Concurrency
- **State:** Effectively immutable. All defining parameters (origin, direction, dimensions) are set via the constructor and are not modified during the object's lifetime. It is a pure data-holding object.

- **Thread Safety:** This class is inherently thread-safe. As an immutable object with no external dependencies, a single instance can be safely read and processed by multiple world-generation worker threads simultaneously without any need for locks or synchronization. All of its methods are pure functions.

## API Surface
The public API is designed for high-performance spatial queries required during world generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAnchor(vector, t, tv, th) | Vector3d | O(1) | Calculates a precise point on the shape's surface. Used for detailed geometry generation. |
| getWidthAt(t) | double | O(1) | Returns the horizontal radius at a normalized distance `t` (0.0 to 1.0) along the central axis. |
| getHeightAt(t) | double | O(1) | Returns the vertical radius at a normalized distance `t` along the central axis. |
| getProjection(x, z) | double | O(1) | Projects a 2D world coordinate onto the shape's central axis, returning the normalized distance `t`. |
| isValidProjection(t) | boolean | O(1) | Determines if a normalized projection `t` falls within the finite bounds of this segment. |
| getDistanceSq(x, z, t) | double | O(1) | Calculates the squared 2D distance of a point from the central axis. Optimized for fast inside/outside checks. |

## Integration Patterns

### Standard Usage
The class is intended to be used by a voxelization kernel. A generator creates the shape via its factory, then passes the resulting object to a processor that carves out blocks in the world.

```java
// Conceptual usage within a world generator
DistortedShape.Factory shapeFactory = new DistortedCylinderShape.Factory();
Vector3d startNode = new Vector3d(100, 45, 250);
Vector3d endNode = new Vector3d(150, 55, 240);
Vector3d direction = endNode.sub(startNode);

// The factory correctly applies pitch compensation before instantiation
DistortedShape caveSegment = shapeFactory.create(
    startNode, direction, direction.length(),
    4.0, 4.0, // start width/height
    5.5, 6.0, // mid width/height
    3.0, 3.5, // end width/height
    GeneralNoise.InterpolationFunction.CUBIC
);

// A Voxelizer would then use this object to determine which blocks to carve
// voxelizer.processChunk(chunk, caveSegment);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DistortedCylinderShape(...)` directly. Doing so bypasses the mandatory `Factory`, which contains the pitch compensation logic. Failure to use the factory will result in visually jarring artifacts where sloped caves appear flattened.

- **State Modification:** Do not attempt to modify the internal `o` (origin) or `v` (direction) vectors after the object has been created. Although the Vector3d class is mutable, the DistortedCylinderShape assumes these values are constant. Modifying them post-construction will lead to unpredictable and incorrect geometric calculations.

## Data Pipeline
This class acts as a configuration model within a larger data processing pipeline for world generation. It does not transform data itself but provides the rules for a transformation.

> Flow:
> Cave Network Graph -> **DistortedCylinderShape.Factory** -> **DistortedCylinderShape Instance** -> Voxelization Kernel -> Modified Chunk Block Data

