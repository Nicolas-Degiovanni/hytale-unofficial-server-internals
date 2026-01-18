---
description: Architectural reference for the CaveNodeShape interface, the core contract for procedural cave geometry.
---

# CaveNodeShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface CaveNodeShape {
    // ... methods
}
```

## Architecture & Concepts
The CaveNodeShape interface is a foundational contract within the server-side procedural world generation engine. It embodies the Strategy Pattern to define the specific geometric volume and population logic for a single segment, or node, of a cave system.

This interface decouples the high-level cave network graph, represented by Cave and CaveNode objects, from the low-level mathematics of how a cave segment is shaped. By providing a concrete implementation of CaveNodeShape, developers can define varied cave geometries—such as spherical chambers, winding tunnels, or vertical shafts—without altering the core cave generation orchestrator.

Its primary role is to answer two fundamental questions for the ChunkGeneratorExecution:
1.  Given a world coordinate, is it inside this cave segment's volume? (shouldReplace)
2.  How should the volume defined by this shape be populated with blocks and features? (populateChunk)

The IWorldBounds returned by getBounds is a critical optimization, allowing the world generator to quickly cull entire chunks that do not intersect with the cave segment, drastically reducing computational overhead.

## Lifecycle & Ownership
As an interface, CaveNodeShape itself has no lifecycle. The lifecycle described here pertains to its concrete implementations.

-   **Creation:** Implementations are created transiently by the cave generation system, typically by a factory or builder when a CaveNode is being processed. They are instantiated with all necessary geometric parameters (e.g., start point, end point, radius) at the moment of creation.
-   **Scope:** The lifetime of a CaveNodeShape implementation is extremely short. It exists only for the duration of the generation process for a specific CaveNode and the chunks it intersects. It is considered an ephemeral, task-scoped object.
-   **Destruction:** The object is eligible for garbage collection as soon as the orchestrating generator has finished processing the corresponding CaveNode across all relevant chunks. There is no manual destruction or cleanup required.

## Internal State & Concurrency
-   **State:** The interface contract implies that implementations should be effectively immutable after creation. All geometric properties (start, end, radius, etc.) are expected to be finalized in the constructor. The methods operate on this initial state and the input parameters.
-   **Thread Safety:** Implementations of this interface **must be thread-safe**. The world generation system is heavily multi-threaded, with different chunks being generated in parallel. A single CaveNodeShape instance may be accessed concurrently by multiple threads processing adjacent chunks. All methods must be pure functions of their inputs and the object's initial, immutable state.

**WARNING:** Introducing mutable shared state into a CaveNodeShape implementation will lead to severe, non-deterministic world generation bugs and race conditions. Any required randomness must be derived from the Random instance provided to populateChunk.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getStart() | Vector3d | O(1) | Returns the pre-calculated starting coordinate of the shape's central axis. |
| getEnd() | Vector3d | O(1) | Returns the pre-calculated ending coordinate of the shape's central axis. |
| getBounds() | IWorldBounds | O(1) | Returns the pre-calculated Axis-Aligned Bounding Box (AABB) that fully encloses the shape. Critical for performance. |
| shouldReplace(block, x, y, z) | boolean | O(1) | The core signed-distance function. Determines if a given world coordinate is inside the shape's volume. Must be highly optimized. |
| getFloorPosition(block, x, z) | double | O(1) | Calculates the world Y-coordinate for the floor of the cave at a given X/Z column. |
| getCeilingPosition(block, x, z) | double | O(1) | Calculates the world Y-coordinate for the ceiling of the cave at a given X/Z column. |
| populateChunk(...) | void | O(N) | The primary procedural generation entry point. Called once per chunk that intersects the shape's bounds. Responsible for carving blocks and placing features. |
| hasGeometry() | boolean | O(1) | A default method indicating if this shape carves physical space. Can be overridden for logical-only nodes. |

## Integration Patterns

### Standard Usage
A concrete implementation of CaveNodeShape is created by a higher-level generator and assigned to a CaveNode. The main chunk generator then uses this shape to carve out the world during its execution pass.

```java
// Within a hypothetical CaveSystemGenerator
void processCaveNode(CaveNode node, ChunkGeneratorExecution execution) {
    // A factory creates a specific shape implementation for the node
    CaveNodeShape shape = CaveShapeFactory.createTunnel(node.getStartPos(), node.getEndPos(), node.getRadius());

    // The execution engine uses the shape to populate the chunk
    // This call happens internally within the world generator
    shape.populateChunk(chunk.getSectionIndex(), execution, cave, node, execution.getRandom());
}
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Do not design implementations that change their internal state after construction. This breaks thread-safety and leads to inconsistent world generation.
-   **Inaccurate Bounds:** Returning bounds from getBounds that are smaller than the actual shape will result in chunks of the cave being abruptly cut off. Returning excessively large bounds will severely degrade generation performance.
-   **Heavy Computation in Bounds:** The getBounds method must be a simple field accessor. All bounding box calculations must be performed once in the constructor of the implementation.
-   **Ignoring the Random Instance:** Do not use a global or new Random() instance within populateChunk. Always use the one provided by the execution context to ensure deterministic, seed-based world generation.

## Data Pipeline
The CaveNodeShape is a processor in the procedural generation pipeline. It does not transform data in a traditional sense, but rather uses input parameters to apply a geometric algorithm to a chunk's block data.

> Flow:
> CaveSystemGenerator -> Creates CaveNode with position/radius data -> Instantiates **CaveNodeShape** implementation -> ChunkGeneratorExecution (for a specific chunk) -> Queries **getBounds()** for culling -> Iterates internal block volume -> Calls **shouldReplace()** for each block -> If true, sets block to AIR. -> Calls **populateChunk()** to place ores, details, etc.

