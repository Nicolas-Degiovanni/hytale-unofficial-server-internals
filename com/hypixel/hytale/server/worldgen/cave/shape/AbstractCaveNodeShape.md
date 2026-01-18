---
description: Architectural reference for AbstractCaveNodeShape
---

# AbstractCaveNodeShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Template (Strategy Pattern)

## Definition
```java
// Signature
public abstract class AbstractCaveNodeShape implements CaveNodeShape {
```

## Architecture & Concepts
The AbstractCaveNodeShape class is a foundational component of the procedural cave generation system. It serves as an abstract base class implementing the Strategy design pattern, where concrete subclasses define the specific geometric shape (e.g., sphere, tunnel, chamber) of a single cave segment, known as a CaveNode.

Its primary responsibility is to translate a high-level, abstract CaveNode definition into low-level block modifications within a specific chunk. It acts as the "carver" in the world generation pipeline, operating on a chunk-by-chunk basis. This class is not responsible for the overall graph or network structure of a cave system; it is exclusively focused on the voxelization of a single node's geometry and the application of its associated materials and decorative covers.

The core logic iterates through a bounded 3D volume, calling the abstract method *shouldReplace* for each block. This delegates the geometric decision to the concrete implementation, allowing for a clean separation of concerns between the general carving algorithm and the specific shape logic.

### Lifecycle & Ownership
- **Creation:** Instances of concrete subclasses are created on-demand by the world generation system, likely within a master CaveGenerator. They are not long-lived, injectable services.
- **Scope:** The scope is transient and tightly bound to the processing of a single CaveNode within a single chunk. An instance typically exists only for the duration of the `populateChunk` method call.
- **Destruction:** The object is stateless and holds no external references, making it eligible for garbage collection immediately after the `populateChunk` method completes.

## Internal State & Concurrency
- **State:** This class is designed to be **stateless**. All necessary information, including the target chunk, cave properties, and random seed, is passed as arguments to its methods. Any internal state is confined to local variables within a method's scope. Subclasses are expected to adhere to this stateless contract.

- **Thread Safety:** The class itself is inherently thread-safe. However, it operates upon a `ChunkGeneratorExecution` context, which is **not thread-safe**. World generation is a highly parallelized process where each chunk is typically processed by a single worker thread.

    **WARNING:** An instance of this class must only be used by the thread that owns the `ChunkGeneratorExecution` context it is given. Any cross-thread access to the execution context will result in severe data corruption, race conditions, and unpredictable world generation outcomes.

## API Surface
The public contract is primarily defined by the `CaveNodeShape` interface it implements.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| populateChunk(seed, execution, cave, node, random) | void | O(W*D*H) | The primary entry point. Voxelizes the node's shape into the target chunk. Complexity is proportional to the volume of the shape's bounding box within the chunk. |

## Integration Patterns

### Standard Usage
This class is intended to be used by a higher-level generator responsible for orchestrating the creation of a cave system. The generator selects a concrete implementation based on a CaveNode's type and invokes `populateChunk` as part of its chunk processing pipeline.

```java
// Hypothetical usage within a CaveGenerator
void generateCavesForChunk(ChunkGeneratorExecution execution) {
    List<CaveNode> nodesInChunk = findNodesImpactingChunk(execution.getX(), execution.getZ());

    for (CaveNode node : nodesInChunk) {
        // Factory or switch statement creates the correct shape strategy
        CaveNodeShape shape = createShapeForNode(node);
        Random nodeRandom = new Random(seed + node.getSeedOffset());

        // Delegate the actual block carving to the shape implementation
        shape.populateChunk(seed, execution, node.getParentCave(), node, nodeRandom);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Concrete subclasses must not store state across multiple `populateChunk` calls. Doing so violates the class's contract and will cause unpredictable, non-deterministic behavior in a multi-threaded environment.
- **Cross-Chunk Modification:** The implementation must never attempt to read or write block data outside the bounds of the provided `ChunkGeneratorExecution` context. The base class's boundary calculations are designed to prevent this, and circumventing them will lead to world corruption.
- **Reusing Instances:** Do not cache and reuse instances of this class across different worker threads. While the object itself is stateless, this practice is dangerous and provides no performance benefit due to the object's lightweight nature.

## Data Pipeline
AbstractCaveNodeShape is a processor stage within the larger world generation data pipeline. It consumes abstract definitions and produces concrete block data.

> Flow:
> Cave Network Generation -> CaveNode Definition -> **AbstractCaveNodeShape (Voxelization)** -> `GeneratedBlockChunk` Modification -> Final Chunk Composition

