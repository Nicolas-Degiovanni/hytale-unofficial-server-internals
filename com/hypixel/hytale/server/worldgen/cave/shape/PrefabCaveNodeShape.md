---
description: Architectural reference for PrefabCaveNodeShape
---

# PrefabCaveNodeShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Transient

## Definition
```java
// Signature
public class PrefabCaveNodeShape implements CaveNodeShape, IWorldBounds {
```

## Architecture & Concepts
The PrefabCaveNodeShape is a concrete implementation of the CaveNodeShape interface, designed to represent a segment of a cave system that is stamped into the world from a pre-designed structure, known as a prefab. Unlike procedurally generated shapes like worms or spheres which are defined by mathematical functions, this class uses discrete, authored data.

It serves as a critical data-holding object that bridges the abstract cave graph system with the concrete block-pasting machinery of the world generator. Each instance encapsulates all necessary information to place a specific prefab at a precise location and orientation:
*   The source prefab data, accessed via a WorldGenPrefabSupplier.
*   The world-space origin vector.
*   A specific PrefabRotation.
*   A BlockMaskCondition to control which existing blocks are replaced.

By also implementing the IWorldBounds interface, this class provides an essential optimization for the world generator. The generator can quickly query the pre-calculated bounding box of the prefab to determine which chunks it intersects, ensuring that the expensive `populateChunk` logic is only executed for relevant chunks.

## Lifecycle & Ownership
- **Creation:** An instance of PrefabCaveNodeShape is created exclusively by its corresponding factory, the inner class PrefabCaveNodeShapeGenerator. This generator is invoked by the core cave generation system when it processes a CaveNode that requires a prefab-based shape. Manual instantiation is a design violation.
- **Scope:** The object's lifetime is ephemeral and tied to a specific world generation task. It is created during the cave layout phase, referenced by any ChunkGeneratorExecution tasks that overlap its bounds, and becomes eligible for garbage collection once those chunk generation tasks are complete. It does not persist in memory after its region is generated.
- **Destruction:** Cleanup is managed by the standard Java garbage collector. There are no native resources or explicit destruction methods.

## Internal State & Concurrency
- **State:** The PrefabCaveNodeShape is **effectively immutable**. All of its defining properties, including the origin vector, prefab supplier, and rotation, are set once in the constructor. The world bounds are also calculated upon instantiation and cached in final fields. This design guarantees that a shape's definition cannot change during its lifetime.
- **Thread Safety:** This class is **thread-safe for all read operations**. Due to its immutable nature, a single instance can be safely shared and read by multiple chunk generation worker threads simultaneously without locks or synchronization. The primary `populateChunk` method mutates the state of the passed-in ChunkGeneratorExecution object, which is thread-local to the worker, thereby preventing race conditions.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| populateChunk(seed, execution, cave, node, random) | void | O(N) | The primary execution method. Pastes the prefab into the target chunk buffer. Complexity is proportional to the number of blocks (N) in the prefab that intersect the chunk. |
| getBounds() | IWorldBounds | O(1) | Returns a reference to itself, providing the pre-calculated world-space bounding box. |
| getFloorPosition(seed, x, z) | double | O(1) | Queries the underlying prefab data to find the lowest Y-coordinate of the shape at a given world X/Z. |
| getCeilingPosition(seed, x, z) | double | O(1) | Queries the underlying prefab data to find the highest Y-coordinate of the shape at a given world X/Z. |
| getStart() | Vector3d | O(1) | Returns the origin point of the shape. |
| getEnd() | Vector3d | O(1) | Returns the end point of the shape, calculated from its origin and extent. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. Instead, it is used implicitly by the world generation engine. A developer configures a CaveType to use the PrefabCaveNodeShapeGenerator, and the engine handles the lifecycle. The engine's chunk generator then invokes `populateChunk` on the instance for each relevant chunk.

```java
// Conceptual view from within a ChunkGeneratorExecution task
void generateCavesForChunk(ChunkGeneratorExecution execution) {
    // The engine first finds all shapes intersecting this chunk
    List<CaveNodeShape> intersectingShapes = findIntersectingShapes(execution.getBounds());

    for (CaveNodeShape shape : intersectingShapes) {
        // The engine dispatches to the correct implementation.
        // If 'shape' is a PrefabCaveNodeShape, this method is called.
        shape.populateChunk(seed, execution, cave, node, random);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PrefabCaveNodeShape(...)`. The object is complex to configure correctly and must be created by its designated `PrefabCaveNodeShapeGenerator` to ensure it is properly integrated into the cave graph with correct rotations and offsets.
- **State Modification:** Do not attempt to modify the internal state of this object via reflection. Its immutability is a core design principle that guarantees thread safety during world generation. Modifying its origin or bounds post-creation will lead to severe and difficult-to-diagnose generation artifacts.
- **Ignoring shouldReplace:** The `shouldReplace` method currently returns false, indicating that the pasting logic is handled entirely within `populateChunk` via the `BlockMaskCondition`. Do not build external logic that assumes this method will perform carving operations.

## Data Pipeline
The flow of data from abstract configuration to concrete block placement is as follows.

> Flow:
> Cave Graph Generation -> Creates CaveNode of prefab type -> Invokes **PrefabCaveNodeShapeGenerator** -> Instantiates **PrefabCaveNodeShape** -> Chunk Generator queries IWorldBounds -> Calls **populateChunk** -> PrefabPasteUtil -> Writes to Chunk Block Buffer

---

