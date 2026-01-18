---
description: Architectural reference for CylinderCaveNodeShape
---

# CylinderCaveNodeShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Transient Data Object

## Definition
```java
// Signature
public class CylinderCaveNodeShape extends AbstractCaveNodeShape implements IWorldBounds {
```

## Architecture & Concepts
The CylinderCaveNodeShape is a fundamental geometric primitive within the server-side procedural cave generation system. It represents a single, directional segment of a cave tunnel. Architecturally, it serves as an immutable data object that defines a volumetric space to be carved out of the world.

Unlike a perfect mathematical cylinder, this shape is more complex, defined by a start radius, an end radius, and a middle radius. This allows for the creation of tunnels that can bulge, pinch, or taper, resulting in more organic and visually interesting cave formations.

Its primary role is to answer the spatial query: "Is a given block coordinate inside this cave segment?". The world generation engine uses this information to replace solid blocks (like stone) with air, effectively creating the cave. The class pre-calculates its own axis-aligned bounding box (AABB) upon instantiation, a critical optimization that allows the generator to query only a small, relevant volume of the world instead of an entire chunk.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly. They are procedurally instantiated by a corresponding factory, **CylinderCaveNodeShapeGenerator**, during the cave graph generation phase. This generator uses randomness and contextual rules (such as the radius of the parent node) to define the shape's final parameters.
-   **Scope:** Extremely short-lived and transient. A CylinderCaveNodeShape object exists only within the memory scope of a single world generation task for a specific region. It is not serialized, persisted, or referenced after the relevant blocks have been modified.
-   **Destruction:** The object is eligible for garbage collection as soon as the generation process that created it completes and discards its reference. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** Immutable. All definitional fields (origin vector, direction vector, radii, and bounds) are final and set exclusively within the constructor. This design is intentional and critical for the multithreaded world generation environment.
-   **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that it can be safely read by multiple worker threads simultaneously without locks or synchronization. This is a core requirement for the parallel processing of world chunks during generation.

    **WARNING:** While the class itself is immutable, it holds references to Vector3d objects. The underlying Vector3d class is mutable. However, CylinderCaveNodeShape mitigates this risk by returning clones or new instances from its public API (e.g., getStart, getEnd), preventing external mutation of its internal state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldReplace(seed, x, z, y) | boolean | O(1) | Core method. Determines if the specified world coordinate is inside the volume of this shape. |
| getBounds() | IWorldBounds | O(1) | Returns the pre-computed bounding box for this shape. Essential for spatial query optimization. |
| getStart() | Vector3d | O(1) | Returns a clone of the starting point of the cylinder's central axis. |
| getEnd() | Vector3d | O(1) | Calculates and returns the end point of the cylinder's central axis. |
| getFloorPosition(seed, x, z) | double | O(N) | Scans vertically to find the lowest point of the cave volume at a given column. N is vertical height. |
| getCeilingPosition(seed, x, z) | double | O(N) | Scans vertically to find the highest point of the cave volume at a given column. N is vertical height. |

## Integration Patterns

### Standard Usage
The class is intended to be used by a higher-level world generation service. The generator obtains a shape instance from its factory and uses it as a spatial predicate to modify a chunk data buffer.

```java
// Pseudo-code for a world generator service
CylinderCaveNodeShapeGenerator generator = getGeneratorFor(caveType);
CaveNodeShape shape = generator.generateCaveNodeShape(...);

IWorldBounds bounds = shape.getBounds();
for (int y = bounds.getLowBoundY(); y <= bounds.getHighBoundY(); y++) {
    for (int z = bounds.getLowBoundZ(); z <= bounds.getHighBoundZ(); z++) {
        for (int x = bounds.getLowBoundX(); x <= bounds.getHighBoundX(); x++) {
            // The core interaction: query the shape
            if (shape.shouldReplace(worldSeed, x, z, y)) {
                chunkBuffer.setBlock(x, y, z, Blocks.AIR);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new CylinderCaveNodeShape(...)` directly. Doing so circumvents the procedural generation logic, which is designed to create varied and context-aware shapes. Always use the appropriate **CylinderCaveNodeShapeGenerator**.
-   **Ignoring Bounds:** Iterating over an entire chunk and calling shouldReplace for every block is computationally expensive. Always use the getBounds method first to drastically reduce the number of checks required.
-   **State Mutation:** Do not attempt to modify the internal vectors of the shape via reflection or other means. The system relies on the immutability of these objects for thread safety.

## Data Pipeline
The CylinderCaveNodeShape is a key component in the data transformation pipeline that turns abstract rules into a modified world.

> Flow:
> Cave Generation Rules (JSON/Config) -> **CylinderCaveNodeShapeGenerator** (Factory) -> Procedural Instantiation -> **CylinderCaveNodeShape** (In-memory representation) -> World Generator Service -> `shouldReplace()` query -> Block ID update in Chunk Buffer -> Finalized Chunk Data

