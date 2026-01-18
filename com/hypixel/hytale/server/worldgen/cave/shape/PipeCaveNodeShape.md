---
description: Architectural reference for PipeCaveNodeShape
---

# PipeCaveNodeShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Transient

## Definition
```java
// Signature
public class PipeCaveNodeShape extends AbstractCaveNodeShape implements IWorldBounds {
```

## Architecture & Concepts
The PipeCaveNodeShape is a fundamental geometric primitive within the server-side procedural cave generation system. It represents a single, straight segment of a cave tunnel, mathematically defined as a cylinder with a linearly interpolated radius from its start to its end point.

Its primary architectural role is to serve as a spatial query object. The world generation engine uses instances of this class to make a binary decision for any given coordinate in the world: "Is this block inside the cave volume, or outside?" This function is encapsulated in the core `shouldReplace` method.

As a concrete implementation of AbstractCaveNodeShape, it is designed to be a component in a larger, graph-like structure of CaveNodes that form a complete cave network. The implementation of the IWorldBounds interface is a critical performance optimization. It allows the world generator to perform a fast, broad-phase check using an Axis-Aligned Bounding Box (AABB), culling entire world regions that do not intersect with the shape before proceeding to more expensive per-block checks.

### Lifecycle & Ownership
-   **Creation:** An instance is never created directly. It is instantiated exclusively by its corresponding factory, the nested PipeCaveNodeShapeGenerator, during the cave network layout phase of world generation. This generator uses procedural rules and a Random source to define the shape's dimensions and orientation.
-   **Scope:** The object's lifetime is extremely short and confined to the generation of a specific cave system. It is a value object that holds the geometric definition of a tunnel segment while the generator carves it into the world's block data.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the world generator has processed the region defined by its bounds. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields, including the origin and direction vectors, radii, and the pre-calculated bounding box, are set once in the constructor and never modified. This design guarantees that the geometric representation of the cave segment is constant and predictable throughout its lifecycle.

-   **Thread Safety:** **Fully thread-safe**. Its immutable nature makes it inherently safe for concurrent access. Multiple world generation worker threads can read from the same PipeCaveNodeShape instance without any risk of data corruption or race conditions. No synchronization or locking mechanisms are required or used.

## API Surface
The public API is designed for high-performance spatial queries by the world generation system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldReplace(seed, x, z, y) | boolean | O(1) | **Core Method.** Determines if a world coordinate is inside the pipe volume. This is the primary query used for carving terrain. |
| getBounds() | IWorldBounds | O(1) | Returns the pre-calculated Axis-Aligned Bounding Box. Essential for broad-phase culling. |
| getStart() | Vector3d | O(1) | Returns the coordinate of the pipe's starting point. |
| getEnd() | Vector3d | O(1) | Returns the coordinate of the pipe's ending point. |
| getFloorPosition(seed, x, z) | double | O(N) | Performs a vertical raycast from the bottom of the bounds to find the lowest point of the cave shape at a given column. N is the height of the bounding box. |
| getCeilingPosition(seed, x, z) | double | O(N) | Performs a vertical raycast from the top of the bounds to find the highest point of the cave shape at a given column. N is the height of the bounding box. |

## Integration Patterns

### Standard Usage
The canonical use involves the world generator obtaining a shape from a generator, using its bounds to identify a work area, and then iterating within that area to test each block.

```java
// Conceptual example within a world generator
PipeCaveNodeShape caveSegment = generator.generateCaveNodeShape(...);
IWorldBounds bounds = caveSegment.getBounds();

// Iterate only over the blocks within the shape's bounding box
for (int y = bounds.getLowBoundY(); y < bounds.getHighBoundY(); y++) {
    for (int z = bounds.getLowBoundZ(); z < bounds.getHighBoundZ(); z++) {
        for (int x = bounds.getLowBoundX(); x < bounds.getHighBoundX(); x++) {
            
            // The core query to decide if a block should be carved
            if (caveSegment.shouldReplace(worldSeed, x, z, y)) {
                world.setBlock(x, y, z, Block.AIR);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new PipeCaveNodeShape()`. Doing so bypasses the procedural generation rules managed by PipeCaveNodeShapeGenerator, which can lead to malformed or disconnected cave systems that violate the world's design constraints. Always rely on the registered generator.
-   **Ignoring Bounds:** Failing to use the `getBounds()` method for a broad-phase check before calling `shouldReplace` is a severe performance anti-pattern. The system is designed around this two-phase query process; skipping the first phase will force the generator to perform millions of unnecessary and expensive geometric calculations across the entire world.

## Data Pipeline
PipeCaveNodeShape acts as a processing node that transforms procedural parameters into a concrete spatial volume used for world modification.

> Flow:
> CaveNodeShapeGenerator -> (Procedural Rules, Random Seed, Parent Node) -> **PipeCaveNodeShape Instance** -> World Generator -> `shouldReplace(x,y,z)` -> Boolean Decision -> World Block Data Mutation

---
# PipeCaveNodeShape.PipeCaveNodeShapeGenerator

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Factory

## Definition
```java
// Signature
public static class PipeCaveNodeShapeGenerator implements CaveNodeShapeEnum.CaveNodeShapeGenerator {
```

## Architecture & Concepts
This class is a factory responsible for the procedural instantiation of PipeCaveNodeShape objects. It encapsulates the rules and logic for creating a new cave segment, such as its length, radius, and orientation, based on a set of configurable parameters and the state of its parent node in the cave network.

Its role is to decouple the geometric definition of a pipe (PipeCaveNodeShape) from the procedural logic used to create it. This allows world designers to configure different types of pipe-based caves by simply providing different generator configurations (e.g., long and thin pipes, short and wide pipes) without altering the core geometric class.

### Lifecycle & Ownership
-   **Creation:** Instantiated once during the server's bootstrap phase when the world generation systems are being configured. A single generator instance is typically created for each variant of pipe-like cave defined in the game's content files.
-   **Scope:** Application-scoped. A generator instance persists for the entire lifetime of the server and is used repeatedly across all world generation tasks.
-   **Destruction:** Destroyed only when the server shuts down.

## Internal State & Concurrency
-   **State:** **Immutable**. The generator's fields (IDoubleRange for radius/length, inheritParentRadius flag) are configured at creation and are read-only thereafter.
-   **Thread Safety:** **Fully thread-safe**. As an immutable factory, it can be safely used by multiple world generation threads concurrently to generate new cave shapes without any external locking. The Random object passed into the `generateCaveNodeShape` method is expected to be thread-local or otherwise managed by the calling thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateCaveNodeShape(...) | CaveNodeShape | O(1) | The sole factory method. Takes all contextual information and a Random source to produce a new, fully-initialized PipeCaveNodeShape instance. |

## Integration Patterns

### Standard Usage
The generator is retrieved from a central registry (like CaveNodeShapeEnum) and used by the main cave generation algorithm to extend the cave network from an existing node.

```java
// Conceptual example within the cave network builder
CaveNode parentNode = ...;
CaveNodeType.CaveNodeChildEntry childEntry = ...; // Defines rules for the child
Random threadLocalRandom = ...;

// Get the appropriate generator for this child type
CaveNodeShapeEnum.CaveNodeShapeGenerator generator = childEntry.getShape().getGenerator();

// Generate the next segment of the cave
PipeCaveNodeShape newSegment = (PipeCaveNodeShape) generator.generateCaveNodeShape(
    threadLocalRandom,
    caveType,
    parentNode,
    childEntry,
    parentNode.getEnd(), // Start the new pipe at the end of the parent
    yaw,
    pitch
);
```

