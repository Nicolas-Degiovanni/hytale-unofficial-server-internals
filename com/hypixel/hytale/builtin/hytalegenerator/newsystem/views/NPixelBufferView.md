---
description: Architectural reference for NPixelBufferView
---

# NPixelBufferView

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.views
**Type:** Transient

## Definition
```java
// Signature
public class NPixelBufferView<T> implements VoxelSpace<T> {
```

## Architecture & Concepts

The NPixelBufferView is a specialized **Adapter** that presents a 2D data structure, the NBufferBundle, as a 3D VoxelSpace. Its primary role is to act as a compatibility layer, allowing systems designed to operate on a 3D grid (VoxelSpace) to read and write data from a collection of 2D buffers, such as those used for heightmaps, biome maps, or other planar world generation data.

Architecturally, this class is a lightweight, non-owning view. It does not store any voxel data itself. Instead, it holds a reference to an NBufferBundle.Access.View and performs on-the-fly coordinate translation for every get or set operation.

The core concept is the translation between two coordinate systems:
1.  **Voxel Grid:** The public-facing 3D coordinate system exposed by the VoxelSpace interface.
2.  **Buffer Grid:** The internal 2D coordinate system used by the NBufferBundle to organize its constituent NPixelBuffer chunks.

A critical design constraint, enforced by an assertion in the constructor, is that this view represents a flat, 2D plane as a 3D space with a thickness of one block. All operations are projected onto the Y=0 plane. This makes it fundamentally a 2D-to-3D adapter, not a true 3D volume view. Any attempt to access coordinates outside of Y=0 will fail bounds checks.

## Lifecycle & Ownership

-   **Creation:** NPixelBufferView is instantiated by higher-level world generation systems that need to bridge the gap between 2D buffer data and 3D voxel manipulation logic. It is constructed with a reference to an existing and fully initialized NBufferBundle.Access.View.
-   **Scope:** This object is intended to be **short-lived and transient**. Its lifetime is typically scoped to a single, specific task, such as a generator populating a biome map. It does not own the underlying buffer data; it merely borrows access to it.
-   **Destruction:** The object is eligible for garbage collection as soon as all references to it are dropped. It manages no native resources and has no explicit `destroy` or `close` method. The lifecycle of the underlying data is managed by the NBufferBundle from which the view was created.

## Internal State & Concurrency

-   **State:** The NPixelBufferView is stateful, but its state is immutable after construction. It stores the provided buffer accessor, the voxel type, and the calculated bounds of the view. It does not cache any voxel data. All read/write operations are passed through directly to the underlying NBufferBundle.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locks or synchronization primitives. The thread safety of read and write operations is entirely dependent on the guarantees provided by the underlying NBufferBundle and NPixelBuffer implementations. Concurrent writes to the same coordinates through this view, or any other view on the same data, will result in a race condition.

    **WARNING:** All access to an NPixelBufferView from multiple threads must be externally synchronized. Failure to do so will lead to data corruption and non-deterministic behavior.

## API Surface

The public API is defined by the VoxelSpace interface, but many methods are intentionally unsupported. The focus is on single-voxel random access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getContent(x, y, z) | T | O(1) | Retrieves the pixel content at the given world coordinate. Throws an assertion error if the position is out of bounds, including any Y-level other than 0. |
| set(value, x, y, z) | boolean | O(1) | Sets the pixel content at the given world coordinate. Throws an assertion error if the position is out of bounds. |
| isInsideSpace(x, y, z) | boolean | O(1) | Performs a fast check to determine if the coordinate is within the view's X, Z, and Y=0 bounds. |
| forEach(action) | void | - | **Unsupported.** Throws UnsupportedOperationException. This view is not designed for bulk iteration. |
| pasteFrom(source) | void | - | **Unsupported.** Throws UnsupportedOperationException. Bulk data modification is not permitted through this view. |

## Integration Patterns

### Standard Usage

The intended use is to acquire a view from a data context, perform targeted read/write operations, and then discard it. It is a tool for procedural generators to interact with 2D data layers using a 3D coordinate system.

```java
// Assume 'generatorContext' provides access to the world's data bundles.
// This view might represent a biome map.
NBufferBundle.Access.View biomeBufferAccess = generatorContext.getBiomeBufferAccess();
NPixelBufferView<Biome> biomeView = new NPixelBufferView<>(biomeBufferAccess, Biome.class);

// Read the biome at a specific world location (Y is ignored and treated as 0)
Vector3i worldPosition = new Vector3i(150, 0, -230);
Biome currentBiome = biomeView.getContent(worldPosition);

// Modify the biome at that location
if (currentBiome == Biomes.PLAINS) {
    biomeView.set(Biomes.FOREST, worldPosition);
}
```

### Anti-Patterns (Do NOT do this)

-   **Long-Term Storage:** Do not hold references to an NPixelBufferView for longer than the duration of a specific task. It is a transient view, not a persistent data container.
-   **Assuming 3D Volume:** Do not attempt to read or write data at any Y-level other than 0. The view is a 2D plane presented as a 3D slice, and out-of-bounds access will fail.
-   **Using Unsupported Operations:** Do not call methods like `forEach` or `pasteFrom`. They are explicitly disabled and will throw an UnsupportedOperationException, crashing the process. If you need to perform bulk operations, operate on the underlying NBufferBundle directly.
-   **Concurrent Unsynchronized Access:** Do not share an NPixelBufferView across multiple threads without implementing external locking. This will corrupt the underlying buffer data.

## Data Pipeline

The NPixelBufferView acts as a dispatcher in a data access pipeline. It does not process data itself but rather translates and forwards requests to the appropriate underlying data store.

> **Flow for a `getContent` call:**
>
> VoxelSpace API call `getContent(150, 0, -230)` -> **NPixelBufferView** -> `GridUtils.toBufferGrid_fromVoxelGrid` (translates world coordinate to a buffer chunk coordinate) -> `NBufferBundle.Access.getBuffer` (retrieves the correct NPixelBuffer chunk) -> `GridUtils.toVoxelGridInsideBuffer_fromWorldGrid` (translates world coordinate to a local coordinate within the chunk) -> `NPixelBuffer.getPixelContent` (reads from internal array) -> Return Value
---

