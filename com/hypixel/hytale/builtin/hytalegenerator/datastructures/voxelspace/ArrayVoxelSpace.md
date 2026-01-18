---
description: Architectural reference for ArrayVoxelSpace
---

# ArrayVoxelSpace<T>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.voxelspace
**Type:** Transient

## Definition
```java
// Signature
public class ArrayVoxelSpace<T> implements VoxelSpace<T> {
```

## Architecture & Concepts
The ArrayVoxelSpace is a foundational data structure for world generation and manipulation, providing a high-performance, in-memory representation of a dense, three-dimensional grid of voxels. It is the engine's primary tool for handling bounded, rectangular volumes of block data, such as schematics, procedural generation chunks, or structure templates.

The core architectural principle is the mapping of a 3D coordinate space onto a flat, one-dimensional array. This design choice optimizes for memory locality and cache performance, enabling extremely fast sequential access and iteration. The conversion from a world coordinate (x, y, z) to a linear array index is handled internally by the private `arrayIndex` method, abstracting this complexity from the consumer. The mapping formula is `y + x * sizeY + z * sizeY * sizeX`, indicating that data is laid out in memory primarily along the Y-axis, then X, then Z.

A critical concept is the **origin** coordinate. The origin acts as a translation vector, decoupling the public-facing world coordinates from the internal 0-indexed array. This allows an ArrayVoxelSpace to represent a region of the world that does not start at (0, 0, 0) without wasting memory or requiring complex index calculations by the caller. For example, a 10x10x10 space can represent the world volume from (100, 50, 200) to (110, 60, 210) by setting its origin appropriately. All coordinate-based API calls operate in this translated world space.

For performance-critical scenarios involving repeated use, the class features an optional **fast reset** mechanism. When enabled, it pre-allocates a template array. The `fastReset` method then uses the highly optimized `System.arraycopy` to overwrite the entire voxel space with the template's contents, which is significantly faster than a manual, element-by-element iteration.

## Lifecycle & Ownership
- **Creation:** An ArrayVoxelSpace is a transient data object. It is instantiated directly via its public constructors (`new ArrayVoxelSpace(...)`) by systems that need to operate on a bounded grid of data. Common creators include world generators, structure placement services, and schematic parsers.

- **Scope:** The lifetime of an instance is typically short and confined to the scope of the operation that created it. It is designed to be used as a temporary buffer or an in-memory representation of a structure, not as a long-term container for world data.

- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as all references to it are released. It does not hold any native resources and has no explicit `destroy` or `close` method.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable**. The primary state is the `contents` array, which holds the voxel data. This array is directly modified by methods like `set`, `pasteFrom`, and `fastReset`. The `origin` coordinate is also mutable via the `setOrigin` method, which effectively relocates the entire coordinate system of the space.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. Concurrent read and write operations on the same instance from multiple threads will result in data races, memory visibility issues, and undefined behavior. If an ArrayVoxelSpace must be shared between threads, all access to it **must** be protected by external synchronization, such as locks or a synchronized wrapper.

## API Surface
The public API provides direct, low-level access to the voxel grid.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getContent(x, y, z) | T | O(1) | Retrieves the voxel at the specified world coordinates. Throws IndexOutOfBoundsException if the coordinate is outside the defined space. |
| set(content, x, y, z) | boolean | O(1) | Sets the voxel at the specified world coordinates. Returns false if the coordinate is out of bounds, preventing an exception. |
| isInsideSpace(x, y, z) | boolean | O(1) | Performs a bounds check against the world coordinates. This is the primary mechanism for safe access. |
| setOrigin(x, y, z) | void | O(1) | Translates the coordinate system by setting a new origin point. Does not modify voxel data. |
| fastReset() | void | O(N) | Resets the entire space to a pre-configured state using a highly optimized memory copy. Throws IllegalStateException if not configured via `setFastResetTo`. |
| forEach(action) | void | O(N) | Iterates over every voxel within the space, applying the provided consumer action. |

## Integration Patterns

### Standard Usage
The most common use case is to define a bounded area, populate it with data, and then read it back for processing, such as placing it into the world.

```java
// How a developer should normally use this
// 1. Define a 16x16x16 space with its origin at world coordinate (100, 50, 200)
int originX = -100; // Note: origin is inverted from world coordinates
int originY = -50;
int originZ = -200;
ArrayVoxelSpace<BlockID> schematic = new ArrayVoxelSpace<>("MyStructure", 16, 16, 16, originX, originY, originZ);

// 2. Populate the space using world coordinates
BlockID stone = BlockRegistry.get("stone");
schematic.set(stone, 100, 50, 200); // Set a block at the corner

// 3. Iterate and process the data
schematic.forEach((block, x, y, z) -> {
    if (block != null) {
        world.setBlock(block, x, y, z);
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never share a single ArrayVoxelSpace instance across multiple threads without external locking. This is the most common source of errors and data corruption.

- **Representing Sparse Data:** This implementation is optimized for *dense* volumes. Using it to represent a large but mostly empty space (e.g., a 1024x1024x1024 area with only a few blocks) is extremely memory-inefficient. For sparse data, use a more appropriate structure like an Octree or a hash-map-based VoxelSpace.

- **Unchecked `getContent` Calls:** The `getContent` method throws an exception on out-of-bounds access. In performance-critical loops, always prefer to check bounds once with `isInsideSpace` rather than wrapping `getContent` in a try-catch block.

## Data Pipeline
ArrayVoxelSpace acts as a container or a buffer stage within a larger data processing pipeline. It does not actively transform data but holds it in memory for other systems to operate on.

> **World Generation Pipeline:**
> Noise Function -> Biome Selector -> **ArrayVoxelSpace** (as a temporary chunk buffer) -> Structure Placer -> World Chunk

> **Schematic Placement Pipeline:**
> File System -> Schematic Parser -> **ArrayVoxelSpace** (as in-memory representation) -> Structure Placement System -> World Chunk

