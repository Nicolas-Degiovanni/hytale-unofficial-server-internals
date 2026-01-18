---
description: Architectural reference for DistortedCaveNodeShape
---

# DistortedCaveNodeShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Transient

## Definition
```java
// Signature
public class DistortedCaveNodeShape implements CaveNodeShape {
```

## Architecture & Concepts
The DistortedCaveNodeShape is a concrete implementation of the CaveNodeShape strategy interface. Its primary role is to define the volumetric geometry of a single segment or "node"—such as a tunnel or chamber—within a larger cave system during procedural world generation.

This class acts as the execution layer that translates a high-level mathematical description of a shape into low-level block modifications within a game chunk. It encapsulates the complex logic for determining if any given 3D point in world space falls within a procedurally generated and distorted volume.

Architecturally, it is a composition of two key components:
1.  **DistortedShape:** Represents the base geometry, such as a capsule, which defines the core path and radius of the cave segment.
2.  **ShapeDistortion:** A set of noise-based functions that modify the base geometry. This component is responsible for adding organic variation, roughness, and imperfections to the floors, ceilings, and walls, preventing the caves from looking like uniform geometric primitives.

The core operational principle is **procedural carving**. This class does not store an array of blocks. Instead, it provides a mathematical test, shouldReplace, and an application method, populateChunk, which are used by the chunk generator to carve its defined volume out of the existing terrain data.

## Lifecycle & Ownership
-   **Creation:** An instance is created exclusively by its corresponding factory, DistortedCaveNodeShapeGenerator. This generator is itself configured and invoked by the higher-level cave generation system based on CaveType asset definitions. Direct instantiation is an anti-pattern.
-   **Scope:** The lifetime of a DistortedCaveNodeShape instance is extremely short and tied to a specific generation task. It is created to represent a single CaveNode during the population of a specific set of world chunks.
-   **Destruction:** Once the populateChunk method has been executed for all chunks that intersect the shape's bounding box, the object is no longer referenced and becomes eligible for garbage collection. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** This class is effectively **immutable**. Its internal fields, shape and distortion, are final and are injected during construction. It holds no mutable state and performs no internal caching. All calculations are deterministic based on its initial state and the parameters passed to its methods.
-   **Thread Safety:** The class is inherently **thread-safe**. Its immutable design ensures that a single instance can be safely used by multiple chunk generation threads simultaneously without any risk of race conditions or data corruption. This is a critical property for the engine's parallelized world generation architecture.

## API Surface
The public API provides the contract for querying the shape's volume and applying it to a chunk buffer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldReplace(seed, x, z, y) | boolean | O(1) | The primary query method. Determines if a block at a given world coordinate is inside the cave volume. Complexity is dependent on the underlying noise functions. |
| populateChunk(...) | void | O(N³) | The main command method. Iterates over the intersection of its bounds and a chunk's volume, carving out blocks by writing to the ChunkGeneratorExecution buffer. |
| getFloorPosition(seed, x, z) | double | O(1) | Calculates the precise Y-coordinate of the cave floor at a given world column. Returns -1.0 if the column is outside the shape. |
| getCeilingPosition(seed, x, z) | double | O(1) | Calculates the precise Y-coordinate of the cave ceiling at a given world column. Returns -1.0 if the column is outside the shape. |
| getBounds() | IWorldBounds | O(1) | Returns the axis-aligned bounding box of the shape. This is used for broad-phase culling to quickly identify which chunks the shape may intersect. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is an internal component of the world generation pipeline. The engine's cave system selects and configures a DistortedCaveNodeShapeGenerator, which in turn creates a DistortedCaveNodeShape instance. The engine then drives the generation process.

The following conceptual example illustrates how the engine uses the object:

```java
// This code is a conceptual illustration of engine-level integration.
// Developers configure this behavior via asset files, not Java code.

// 1. The generator creates a shape instance for a new cave node.
CaveNodeShape shape = generator.generateCaveNodeShape(random, caveType, ...);

// 2. The engine finds all chunks the shape might touch using its bounds.
for (ChunkCoordinate chunkCoord : shape.getBounds().getIntersectingChunks()) {
    ChunkGeneratorExecution execution = worldGenerator.getExecutionContextFor(chunkCoord);

    // 3. The engine commands the shape to carve itself into the chunk buffer.
    shape.populateChunk(seed, execution, cave, node, random);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new DistortedCaveNodeShape()`. The shape's parameters (size, orientation, distortion) are procedurally determined by its generator to ensure continuity with the parent cave system. Manual creation will result in disconnected or malformed cave segments.
-   **State Modification:** Do not attempt to modify the internal DistortedShape or ShapeDistortion objects after creation, for example, via reflection. This violates the immutability contract and will cause unpredictable and non-deterministic generation results, especially in a multi-threaded context.

## Data Pipeline
This component is a step in a procedural generation pipeline. It transforms a high-level definition into concrete block data.

> Flow:
> CaveType Asset (JSON) → DistortedCaveNodeShapeGenerator (Factory) → **DistortedCaveNodeShape** (In-Memory Representation) → populateChunk Method → ChunkGeneratorExecution (Block Buffer) → Final Chunk Data

