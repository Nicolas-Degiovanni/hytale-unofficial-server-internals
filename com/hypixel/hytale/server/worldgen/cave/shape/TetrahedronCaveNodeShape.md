---
description: Architectural reference for TetrahedronCaveNodeShape
---

# TetrahedronCaveNodeShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Transient

## Definition
```java
// Signature
@Deprecated
public class TetrahedronCaveNodeShape extends AbstractCaveNodeShape implements IWorldBounds {
```

## Architecture & Concepts

The TetrahedronCaveNodeShape is a geometric primitive used within the server-side world generation pipeline to define a volumetric space for cave carving. It represents a fixed-size, tetrahedron-shaped region of the world. Its primary responsibility is to provide a fast, mathematical test to determine if a given world coordinate lies within its volume.

This class is a concrete implementation of the AbstractCaveNodeShape contract and is designed to be a simple, self-contained shape definition. Upon instantiation, it pre-calculates the normal vectors for each of its four faces. These normals are then used in a series of point-plane dot product calculations to perform an efficient inside/outside test.

**WARNING:** This class is annotated as **Deprecated**. It should be considered a legacy component. New development should avoid using this shape in favor of more flexible or performant alternatives provided by the world generation framework. Its fixed size and simplistic geometry limit its utility in creating complex and varied cave networks.

## Lifecycle & Ownership

-   **Creation:** An instance of TetrahedronCaveNodeShape is created exclusively by its corresponding factory, the inner class TetrahedronCaveNodeShapeGenerator. This generator is invoked by the cave generation system when it processes a CaveNode configured to use this specific shape.
-   **Scope:** The object is extremely short-lived. Its scope is confined to the processing of a single cave node during a chunk generation task. It does not persist between sessions or even between distinct generation operations.
-   **Destruction:** The object holds no external resources or references. It becomes eligible for garbage collection immediately after the world generator has used it to carve the relevant voxels in the world data.

## Internal State & Concurrency

-   **State:** **Immutable**. All internal fields, including the origin vector, derived geometric vectors, and pre-calculated bounding box coordinates, are marked as final. The object's state is fully determined at construction and cannot be modified thereafter.
-   **Thread Safety:** This class is inherently **thread-safe**. Its immutability guarantees that it can be safely read and utilized by multiple world generation worker threads simultaneously without any need for external locking or synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldReplace(seed, x, z, y) | boolean | O(1) | **Core Method.** Determines if the specified world coordinate is inside the tetrahedron volume. Returns true if the block should be carved. |
| getBounds() | IWorldBounds | O(1) | Returns a reference to itself, as it implements the bounding box contract. Used for broad-phase collision and culling. |
| getFloorPosition(seed, x, z) | double | O(N) | Scans vertically from the bottom of the bounding box to find the lowest point of the shape. **Warning:** This is a performance-intensive linear scan. |
| getCeilingPosition(seed, x, z) | double | O(N) | Scans vertically from the top of the bounding box to find the highest point of the shape. **Warning:** This is a performance-intensive linear scan. |
| getStart() | Vector3d | O(1) | Returns the origin point of the tetrahedron, used as an anchor. |
| getEnd() | Vector3d | O(1) | Returns a designated endpoint, typically used for connecting subsequent cave segments. |

## Integration Patterns

### Standard Usage

This class is not intended to be used directly. The world generation system interacts with it via its factory, which is typically registered with an enumeration or manager that maps cave node types to shape generators.

```java
// Conceptual example of how the generator is used by the engine
// A developer would NOT write this code.

// 1. The engine retrieves the generator for a given cave node type
CaveNodeShapeEnum.CaveNodeShapeGenerator generator = CaveNodeShapeEnum.TETRAHEDRON.getGenerator();

// 2. The engine invokes the generator to create a shape instance at a specific origin
CaveNodeShape shape = generator.generateCaveNodeShape(random, caveType, parent, child, origin, yaw, pitch);

// 3. The engine uses the shape to carve world data
for (int y = shape.getLowBoundY(); y < shape.getHighBoundY(); y++) {
    if (shape.shouldReplace(seed, x, z, y)) {
        world.setBlock(x, y, z, Blocks.AIR);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new TetrahedronCaveNodeShape(origin)`. The creation is managed entirely by the world generation framework through the `TetrahedronCaveNodeShapeGenerator` factory.
-   **Usage in New Features:** Do not use this class for any new cave types or world generation features. Its deprecated status indicates it is obsolete and may be removed in future versions.
-   **Performance-Critical Loops:** Avoid calling `getFloorPosition` or `getCeilingPosition` in tight loops. Their O(N) complexity can create significant performance bottlenecks during world generation. Prefer using `shouldReplace` within the known bounding box.

## Data Pipeline

The flow of data and control involves this class as a transient, computational step within the larger world generation process.

> Flow:
> World Generator -> Process Cave Node -> **TetrahedronCaveNodeShapeGenerator** -> Instantiate **TetrahedronCaveNodeShape** -> Voxel Carver uses `shouldReplace()` -> Modify Chunk Voxel Buffer

