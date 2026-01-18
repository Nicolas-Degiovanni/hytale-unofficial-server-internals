---
description: Architectural reference for the VoxelSpace interface, the core contract for 3D voxel data structures.
---

# VoxelSpace

**Package:** `com.hypixel.hytale.builtin.hytalegenerator.datastructures.voxelspace`
**Type:** Interface

## Definition
```java
// Signature
public interface VoxelSpace<T> {
    // Key methods defined below
}
```

## Architecture & Concepts

The VoxelSpace interface is a foundational abstraction within the Hytale world generation framework. It establishes a standard contract for any three-dimensional, grid-aligned data structure that stores voxel information. It is not a concrete class, but rather a blueprint that separates procedural generation logic from the underlying data storage mechanism.

This design is critical for performance and flexibility. By programming against the VoxelSpace interface, a generation algorithm can operate on a simple array-backed grid, a memory-efficient sparse octree, or any other implementation without modification. The choice of implementation can be tailored to the specific use case, balancing memory footprint against access speed.

The generic parameter `T` allows a VoxelSpace to be versatile. It can be configured to hold simple block identifiers (e.g., `VoxelSpace<Integer>`) or complex, stateful objects representing block entities or procedural metadata (`VoxelSpace<CustomBlockData>`).

A core concept is the **relative coordinate system**. Every VoxelSpace has an origin (getOriginX, getOriginY, getOriginZ) and bounds (minX, maxX, etc.). All access methods like `set` and `getContent` operate in coordinates relative to this origin. This makes VoxelSpace instances ideal for representing modular, movable structures such as prefabs, schematics, or temporary generation buffers that are later "pasted" into the absolute world coordinate system.

## Lifecycle & Ownership

As an interface, VoxelSpace does not have a lifecycle itself. The following applies to its concrete implementations (e.g., ArrayVoxelSpace, SparseVoxelSpace).

-   **Creation:** Instances are typically created by high-level systems that require a temporary or modular 3D data buffer. Common creators include world generation stage runners, prefab loaders, or gameplay systems that manipulate blocks in a defined volume.
-   **Scope:** The lifetime of a VoxelSpace instance is highly context-dependent. It may be extremely short-lived, existing only for the duration of a single function call to act as a temporary buffer. Conversely, it could be long-lived, representing a loaded prefab that is held in memory and pasted into the world multiple times.
-   **Destruction:** Management is delegated to the Java Garbage Collector. The system that instantiates a VoxelSpace is responsible for releasing all references to it once its purpose is fulfilled. There is no explicit `destroy` or `close` method in the contract.

## Internal State & Concurrency

-   **State:** Any implementation of VoxelSpace is inherently stateful and mutable. It represents a collection of data stored at 3D integer coordinates. Methods like `set`, `replace`, and `pasteFrom` are designed to mutate this internal state.
-   **Thread Safety:** The VoxelSpace contract provides **no guarantee of thread safety**. Implementations are assumed to be non-thread-safe unless explicitly documented otherwise. All concurrent read/write operations on a single VoxelSpace instance must be managed with external synchronization (e.g., locks). This design choice prioritizes single-threaded performance, which is the common case in many generation stages.

**WARNING:** Unsynchronized access to a VoxelSpace from multiple threads will lead to data corruption, race conditions, and undefined behavior. It is the responsibility of the calling system to enforce thread safety.

## API Surface

The public API defines a complete set of operations for manipulating a 3D grid of data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| set(T content, int x, int y, int z) | boolean | O(1) to O(log N) | Places content at the specified relative coordinates. Complexity depends on implementation (O(1) for arrays, O(log N) for trees). |
| getContent(int x, int y, int z) | T | O(1) to O(log N) | Retrieves content from the specified relative coordinates. Returns null if the location is empty or out of bounds. |
| replace(T content, ..., Predicate<T> condition) | boolean | O(1) to O(log N) | Atomically replaces existing content if it matches the predicate. Essential for conditional procedural logic. |
| pasteFrom(VoxelSpace<T> source) | void | O(N) | Performs a bulk copy, pasting the source space into this one. N is the volume of the source space. |
| forEach(VoxelConsumer consumer) | void | O(N) | Iterates over every voxel within the bounds, applying the consumer. N is the total volume of the space. |
| getBounds() | Bounds3i | O(1) | Returns the axis-aligned bounding box of the space in its own relative coordinate system. |

## Integration Patterns

### Standard Usage

The primary use case is as a data buffer during procedural generation. A system creates a VoxelSpace, populates it algorithmically, and then potentially pastes it into a larger world context.

```java
// A hypothetical generator creating a pillar
// NOTE: ArrayVoxelSpace is a hypothetical concrete implementation
VoxelSpace<Integer> pillar = new ArrayVoxelSpace<>(5, 20, 5);
pillar.setOrigin(0, 0, 0); // Set relative origin

// Fill the space with stone (e.g., block ID 1)
pillar.set(1); 

// Carve out the center
for (int y = 0; y < 18; y++) {
    pillar.set(null, 2, y, 2); // Set to "air"
}

// The 'pillar' object can now be passed to other systems
// to be pasted into the world.
world.paste(pillar, new Vector3i(100, 64, 100));
```

### Anti-Patterns (Do NOT do this)

-   **Assuming Thread Safety:** The most critical anti-pattern. Never share and modify a single VoxelSpace instance across multiple threads without explicit, external locking. Instead, provide each thread with its own distinct VoxelSpace instance to work on.
-   **Ignoring the Origin:** Treating coordinates passed to `set` or `getContent` as absolute world coordinates. This will fail when the VoxelSpace is repositioned or used as a prefab. All operations are relative to the space's own origin.
-   **Inefficient Storage Choice:** Using a dense, array-backed implementation for a very large but mostly empty volume (e.g., a 512x512x512 space for a hollow sphere). This will consume an enormous amount of memory. For sparse data, a sparse-octree or hashmap-backed implementation should be used.

## Data Pipeline

VoxelSpace serves as a standardized in-memory representation of voxel data, acting as a crucial intermediate step in generation and construction pipelines.

> Flow:
> Procedural Algorithm (e.g., DungeonGenerator) -> **VoxelSpace<BlockData> (Buffer)** -> World Paster System -> Final Chunk Voxel Data

