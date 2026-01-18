---
description: Architectural reference for EllipsoidCaveNodeShape
---

# EllipsoidCaveNodeShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Transient

## Definition
```java
// Signature
public class EllipsoidCaveNodeShape extends AbstractCaveNodeShape implements IWorldBounds {
```

## Architecture & Concepts
The EllipsoidCaveNodeShape is a fundamental geometric primitive within the server-side procedural world generation engine, specifically for cave systems. It represents a single, immutable ellipsoidal volume in 3D space used to define the shape of a cave chamber or a segment of a tunnel.

This class acts as a concrete implementation of the abstract concept of a CaveNodeShape. The world generator constructs a graph of CaveNode objects, and each node is assigned a shape like this one to define its physical presence. The primary responsibility of this class is to answer a single, critical question: "Is a given coordinate (x, y, z) inside this ellipsoid?" This determination, handled by the shouldReplace method, is the core mechanism for carving caves out of solid terrain.

Crucially, this class also implements the IWorldBounds interface. This provides the generation engine with an axis-aligned bounding box (AABB) for the shape. This is a non-negotiable performance optimization; it allows the engine to rapidly discard entire world chunks or regions that do not intersect with the shape, avoiding expensive per-block checks.

## Lifecycle & Ownership
- **Creation:** EllipsoidCaveNodeShape instances are not intended to be instantiated directly. They are created exclusively by the nested EllipsoidCaveNodeShapeGenerator. This generator is invoked by a higher-level cave generation orchestrator which provides contextual information like the parent cave node, origin point, and random seed. The generator uses this context to calculate randomized radii and final positioning.

- **Scope:** An instance of this class is extremely short-lived. Its scope is tied to the generation of a specific cave segment. It is created, queried thousands of times by the world carver for blocks within its bounds, and then becomes eligible for garbage collection. It does not persist between generation tasks.

- **Destruction:** The object is managed by the Java garbage collector. Once the world carver has finished processing the volume defined by its bounds, all references to the instance are dropped, and it is reclaimed.

## Internal State & Concurrency
- **State:** This object is **effectively immutable**. All of its core state, including its center point, radii, and pre-calculated bounds, is set once in the constructor and never modified. This design guarantees predictable and repeatable geometric queries.

- **Thread Safety:** The immutable nature of this class makes it inherently **thread-safe**. A single EllipsoidCaveNodeShape instance can be safely accessed by multiple world generation threads simultaneously without any need for external locking or synchronization. This is a critical architectural feature for enabling a highly parallelized world generation pipeline.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldReplace(seed, x, z, y) | boolean | O(1) | Core method. Determines if a world coordinate is inside the ellipsoid volume. Incorporates a noise factor via CaveType. |
| getBounds() | IWorldBounds | O(1) | Returns a reference to itself, fulfilling the IWorldBounds contract. |
| getLowBoundY() | int | O(1) | Returns the minimum Y-coordinate of the bounding box. |
| getHighBoundY() | int | O(1) | Returns the maximum Y-coordinate of the bounding box. |
| getFloorPosition(seed, x, z) | double | O(N) | Performs a vertical raycast from the bottom of the bounds upwards to find the first point of intersection. N is the vertical height of the shape. |
| getCeilingPosition(seed, x, z) | double | O(N) | Performs a vertical raycast from the top of the bounds downwards to find the first point of intersection. N is the vertical height of the shape. |

## Integration Patterns

### Standard Usage
The intended use is through a world generation service that leverages the corresponding generator. The service first obtains the shape's bounds to define a work area, then iterates over each block, calling shouldReplace to determine if it should be carved.

```java
// Conceptual example of a world carver using the shape
EllipsoidCaveNodeShape.EllipsoidCaveNodeShapeGenerator generator = getGeneratorForCaveType();
CaveNodeShape shape = generator.generateCaveNodeShape(random, caveType, parent, child, origin, yaw, pitch);

IWorldBounds bounds = shape.getBounds();
for (int y = bounds.getLowBoundY(); y <= bounds.getHighBoundY(); y++) {
    for (int z = bounds.getLowBoundZ(); z <= bounds.getHighBoundZ(); z++) {
        for (int x = bounds.getLowBoundX(); x <= bounds.getHighBoundX(); x++) {
            if (shape.shouldReplace(seed, x, z, y)) {
                world.setBlock(x, y, z, Blocks.AIR);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new EllipsoidCaveNodeShape(...)`. The factory pattern provided by EllipsoidCaveNodeShapeGenerator is essential for applying procedural rules, such as randomized radii and positional offsets relative to a parent node. Bypassing it will lead to disconnected or malformed cave systems.

- **Ignoring Bounds:** Never iterate over an entire chunk and call shouldReplace for every block. Always retrieve the IWorldBounds first and limit the iteration to that volume. Failure to do so will result in a catastrophic performance degradation of the world generator.

- **State Modification:** Do not attempt to modify the internal state of the object via reflection. The immutability of this class is a core assumption of the parallel generation engine.

## Data Pipeline
The EllipsoidCaveNodeShape functions as a geometric predicate within the larger cave generation data pipeline.

> Flow:
> Cave Graph Traversal -> **EllipsoidCaveNodeShapeGenerator** -> Creates **EllipsoidCaveNodeShape** instance -> World Carver queries bounds -> Carver iterates blocks -> Carver calls **shouldReplace()** -> Block data is mutated (e.g., STONE becomes AIR)

