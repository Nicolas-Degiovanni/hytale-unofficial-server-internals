---
description: Architectural reference for CaveNode
---

# CaveNode

**Package:** com.hypixel.hytale.server.worldgen.cave.element
**Type:** Transient

## Definition
```java
// Signature
public class CaveNode implements CaveElement {
```

## Architecture & Concepts
The CaveNode class is a fundamental data structure within the server-side procedural world generation framework, specifically for cave systems. It represents a single, discrete volume or "room" in a larger, interconnected cave network.

Conceptually, a CaveNode acts as a container. It aggregates two primary forms of data:
1.  **Volumetric Shape:** The core geometry of the cave segment is defined by a `CaveNodeShape` object. This shape dictates the floor and ceiling positions, effectively defining the space to be carved out of the world.
2.  **Internal Features:** It holds a collection of `CavePrefab` objects, which represent smaller, self-contained features like stalagmites, crystal formations, or underground ruins that are placed within the node's volume.

A CaveNode is a node in a conceptual graph that forms a complete cave system. A higher-level service, such as a `CaveGraphGenerator`, is responsible for instantiating these nodes, connecting them with tunnels (likely another `CaveElement` implementation), and arranging them to create complex, non-linear underground structures.

## Lifecycle & Ownership
-   **Creation:** A CaveNode is instantiated by a world generation service during the procedural generation phase for a given region. It is not created by the game loop or entity systems. Its initial state is mutable, designed to be configured and populated.

-   **Scope:** The object's lifetime is ephemeral and strictly bound to the world generation task that created it. It exists only in memory while a set of chunks is being generated. It is not serialized or persisted to disk.

-   **Destruction:** Once the world generator has processed the CaveNode—using its data to modify the chunk's block data and place prefabs—all references to the instance are dropped, and it becomes eligible for garbage collection. The `compile` method marks a critical point in its lifecycle, transitioning it from a mutable "builder" state to an immutable, memory-optimized "read-only" state.

## Internal State & Concurrency
-   **State:** The internal state is mutable upon creation. The `addPrefab` method modifies an internal list and recalculates the node's overall bounding box. This mutability is terminated by a call to `compile`, which converts the internal prefab list into a fixed-size array and nullifies the original list. This is a deliberate, one-way state transition designed to prevent further modification and reduce memory footprint.

-   **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access within the context of a specific world generation job. Concurrent calls to `addPrefab` will lead to race conditions and data corruption. Any multi-threaded world generation strategy must ensure that a single CaveNode instance is owned and modified by only one thread at a time.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addPrefab(prefab) | void | O(1) amortized | Adds a prefab to the node's internal collection and expands its bounding box to include it. Throws NullPointerException if called after `compile`. |
| compile() | void | O(N) | Finalizes the node's internal state, making it immutable and ready for consumption by carving and population systems. N is the number of prefabs. |
| getBounds() | IWorldBounds | O(1) | Returns the axis-aligned bounding box that fully encloses the node and all its prefabs. Used for broad-phase culling. |
| getFloorPosition(seed, x, z) | int | O(1) | Delegates to the internal `CaveNodeShape` to calculate the Y-level of the cave floor at a specific world coordinate. |
| getCeilingPosition(seed, x, z) | int | O(1) | Delegates to the internal `CaveNodeShape` to calculate the Y-level of the cave ceiling at a specific world coordinate. |
| forEachChunk(consumer) | void | O(W*D) | Executes a provided callback for every chunk index that this node's bounds intersect. This is a critical optimization primitive. W and D are the width and depth of the node in chunks. |

## Integration Patterns

### Standard Usage
The intended pattern is a two-phase process: build, then consume. A generator service first constructs and populates the CaveNode, then finalizes it with `compile`, and finally passes the immutable object to subsequent systems for application to the world data.

```java
// Within a world generation service...
CaveNodeShape shape = shapeFactory.createSphericalChamber(position, radius);
CaveNode node = new CaveNode(worldSeed, CaveNodeType.CHAMBER, shape, 0.0f, 0.0f);

// Phase 1: Build and Populate
node.addPrefab(new StalagmitePrefab(position.add(5, 0, 2)));
node.addPrefab(new CrystalClusterPrefab(position.add(-3, 0, 8)));

// Phase 2: Finalize and Consume
node.compile();

// Pass the finalized, immutable node to the system that modifies world blocks.
worldCarver.applyElement(node);
```

### Anti-Patterns (Do NOT do this)
-   **Post-Compile Modification:** Do not call `addPrefab` after `compile` has been invoked. This will result in a `NullPointerException` as the internal list used for building is discarded for memory efficiency.
-   **Reusing Instances:** A CaveNode instance is stateful and not designed to be reset or reused. For each new cave chamber or segment, create a new CaveNode instance.
-   **Ignoring Bounding Volumes:** Do not iterate through all blocks in a chunk and check for intersection. Always use `getBounds` and `forEachChunk` to quickly identify which parts of the world the CaveNode might affect. Bypassing these methods will cause catastrophic performance issues.

## Data Pipeline
CaveNode is a data-transfer object that carries information between stages of world generation. It does not actively process data itself but is the subject of processing.

> Flow:
> `CaveGraphGenerator` -> **CaveNode (Instantiation & Population)** -> `compile()` -> **CaveNode (Immutable State)** -> `WorldCarver` / `ChunkPopulator` -> Final Chunk Block Data

