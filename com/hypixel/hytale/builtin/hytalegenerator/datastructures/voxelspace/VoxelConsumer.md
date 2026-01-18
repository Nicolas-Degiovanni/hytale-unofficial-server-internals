---
description: Architectural reference for VoxelConsumer
---

# VoxelConsumer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.voxelspace
**Type:** Behavioral Contract / Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface VoxelConsumer<V> {
```

## Architecture & Concepts

The VoxelConsumer interface is a core component of Hytale's world generation and voxel processing pipeline. It embodies a high-performance, memory-efficient callback pattern for iterating over large volumes of voxel data. Architecturally, it serves as a contract for any system that needs to *process* voxel data without needing to own or manage the underlying storage container.

Instead of a data structure returning a massive, heap-allocated collection of voxels, it implements methods that accept a VoxelConsumer. The data structure then iterates over its internal representation, invoking the consumer's *accept* method for each voxel. This "push" model is fundamentally a form of the Visitor pattern, applied to voxel spaces.

This design achieves several key objectives:
- **Decoupling:** The algorithm that traverses a voxel space (e.g., a chunk or an octree) is completely decoupled from the logic that operates on the individual voxels (e.g., mesh generation, physics simulation, AI analysis).
- **Performance:** It eliminates the significant overhead of creating intermediate collections to hold voxel data, reducing memory allocation pressure and improving cache locality.
- **Flexibility:** Callers can provide unique logic at runtime via lambda expressions, enabling a wide variety of operations on the same underlying data structures without modification.

## Lifecycle & Ownership

- **Creation:** Implementations of VoxelConsumer are almost exclusively created as short-lived lambda expressions or method references at the call site. The entity requesting the voxel traversal is responsible for providing the concrete implementation.
- **Scope:** The lifecycle of a VoxelConsumer instance is ephemeral. It is scoped to the duration of the single method call it is passed into (e.g., `VoxelStorage.forEach`).
- **Destruction:** The object is eligible for garbage collection immediately upon the completion of the traversal method.

**WARNING:** Implementations that capture variables from their enclosing scope may inadvertently extend the lifetime of those captured objects. This is a common source of memory leaks if not managed carefully.

## Internal State & Concurrency

- **State:** The VoxelConsumer interface itself is stateless. However, a specific implementation (i.e., a lambda) can be stateful if it closes over and modifies variables from its surrounding scope. Stateless consumers are strongly preferred as they are predictable and reusable.
- **Thread Safety:** The contract makes no guarantees of thread safety. Safety is a shared responsibility:
    1. **The Data Provider:** The system iterating over the voxel data is responsible for ensuring that it does not invoke the *accept* method from multiple threads simultaneously unless the consumer is explicitly designed for parallel execution.
    2. **The Implementation:** If a consumer is used in a parallel context, its implementation **must** be thread-safe. Modifying a shared, non-synchronized collection like an ArrayList from a consumer used in a parallel stream will result in data corruption and non-deterministic crashes.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(V voxel, int x, int y, int z) | void | O(1) | The single abstract method defining the contract. Invoked by a data provider for each voxel, passing the voxel data and its integer coordinates. |

## Integration Patterns

### Standard Usage

The primary integration pattern is to provide a lambda expression to a method that traverses a voxel data structure. This is the most common and idiomatic use within the engine.

```java
// Assume a VoxelStorage object that holds block data
VoxelStorage<Block> chunk = world.getChunkStorage(chunkPos);
List<Position> waterSources = new ArrayList<>();

// The lambda implements the VoxelConsumer contract to find water blocks.
// This is highly efficient as no intermediate lists of all blocks are created.
chunk.forEachVoxel((block, x, y, z) -> {
    if (block.isWaterSource()) {
        waterSources.add(new Position(x, y, z));
    }
});
```

### Anti-Patterns (Do NOT do this)

- **Blocking Operations:** Do not perform file I/O, network requests, or other long-running, blocking operations inside the *accept* method. The data provider is typically blocked until the consumer returns, and stalling it can cascade into major performance degradation for the entire world-generation pipeline.
- **Stateful Consumers in Parallel Traversal:** Using a consumer that modifies a non-thread-safe external collection during a parallel traversal is a critical error. This will lead to race conditions.

```java
// DANGEROUS: Do not do this.
List<Block> results = new ArrayList<>();
// If forEachVoxelParallel executes the consumer on multiple threads,
// the unsynchronized call to results.add() will corrupt the list.
chunk.forEachVoxelParallel((block, x, y, z) -> {
    if (block.isImportant()) {
        results.add(block); // RACE CONDITION
    }
});
```

## Data Pipeline

VoxelConsumer acts as a terminal processing step in a localized data flow. It is the destination where data from a voxel container is dispatched for domain-specific logic.

> Flow:
> Traversal Request (e.g., `generateMeshForChunk`) -> Voxel Data Structure (e.g., Chunk, Octree) -> **VoxelConsumer.accept()** -> Application Logic (e.g., Mesh Builder, Light Propagator)

