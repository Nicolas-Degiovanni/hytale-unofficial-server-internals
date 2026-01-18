---
description: Architectural reference for BooleanVoxelSpace, a memory-optimized 3D data structure for boolean values.
---

# BooleanVoxelSpace

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.voxelspace
**Type:** Transient

## Definition
```java
// Signature
public class BooleanVoxelSpace implements VoxelSpace<Boolean> {
```

## Architecture & Concepts
BooleanVoxelSpace is a high-performance, memory-optimized data structure designed to represent a three-dimensional volume of boolean values. It serves as a foundational component within the world generation system, typically used for tasks like defining solid versus empty space, marking areas for modification, or representing the footprint of structures.

The core architectural principle is **memory efficiency through bit packing**. Instead of storing each boolean value in a separate byte, it packs 32 boolean values into a single 32-bit integer. This dramatically reduces the memory footprint for large volumes, which is critical for scalable world generation.

This is achieved through a two-tiered addressing scheme:
1.  **Primary Address:** The X and Y coordinates are flattened into a single index to select a primary array or "column" of integers. This column represents all Z values at a given (X, Y) position.
2.  **Secondary Address:** The Z coordinate is used to select a specific integer within that column and a bit index within that integer. All Z-axis operations are performed using bitwise shifts and masks.

A second key concept is the **movable origin**. A BooleanVoxelSpace instance does not assume its coordinates start at (0, 0, 0). Instead, it maintains an internal origin point, allowing it to represent a sub-volume of a much larger world coordinate system without requiring a global data structure. All public API methods operate in world coordinates, which are internally translated into local array indices.

Finally, the class includes special handling for **Z-axis alignment**. The bit-packing strategy is most efficient when operations occur on 32-block boundaries along the Z-axis. The `alignedOriginZ` constructor parameter allows for optimizations when the caller can guarantee this alignment, avoiding costly calculations for unaligned access.

## Lifecycle & Ownership
-   **Creation:** An instance is created directly via its constructor (`new BooleanVoxelSpace(...)`). It is typically instantiated by a world generation algorithm, a procedural content system, or a structure placement service that needs to operate on a bounded 3D volume.
-   **Scope:** The object's lifetime is bound to the scope of the operation that created it. It is a self-contained data object, not a managed service. It may be short-lived for a single calculation or persist longer as part of a larger, more complex data schematic.
-   **Destruction:** The object is eligible for garbage collection once all references to it are released. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The internal state is highly **mutable**. The primary purpose of the class is to modify the `cells` integer array that backs the voxel data. Operations like `set` and `pasteFrom` directly mutate this internal state.

-   **Thread Safety:** **WARNING:** This class is not thread-safe. The internal data structures are accessed without any synchronization mechanisms. Concurrent calls to mutation methods like `set` from multiple threads will lead to race conditions and data corruption, especially when modifying voxels that map to the same underlying integer. All concurrent access must be managed by the caller using external locking.

## API Surface
The public contract is focused on efficient manipulation of boolean data within a 3D volume.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| set(value, x, y, z) | boolean | O(1) | Sets the boolean state of a single voxel. Returns false if coordinates are out of bounds. |
| getContent(x, y, z) | Boolean | O(1) | Retrieves the boolean state of a single voxel. Throws IndexOutOfBoundsException if coordinates are invalid. |
| setOrigin(x, y, z) | void | O(1) | Relocates the coordinate system origin. Throws if Z is unaligned when alignment is enforced. |
| deepCopyFrom(other) | void | O(N) | Performs a highly optimized, block-level copy from another BooleanVoxelSpace. N is the volume of the intersecting space. |
| pasteFrom(source) | void | O(N) | Performs a generic, voxel-by-voxel copy from any VoxelSpace implementation. N is the volume of the source space. |
| isInsideSpace(x, y, z) | boolean | O(1) | Checks if the given world coordinates fall within the defined volume. |

## Integration Patterns

### Standard Usage
This class is intended to be used as a temporary workspace for generation algorithms. An algorithm defines a volume, populates it based on rules or noise functions, and then may use it as a mask or transfer its contents to the primary world data.

```java
// Example: Carving a spherical air pocket into a solid volume.
int size = 64;
BooleanVoxelSpace volume = new BooleanVoxelSpace(size, size, size);

// 1. Initialize the volume to be solid (true)
volume.set(true);

// 2. Carve out a sphere of air (false)
Vector3i center = new Vector3i(size / 2, size / 2, size / 2);
float radius = 20.0f;

for (int x = volume.minX(); x < volume.maxX(); x++) {
    for (int y = volume.minY(); y < volume.maxY(); y++) {
        for (int z = volume.minZ(); z < volume.maxZ(); z++) {
            if (center.distanceSquared(x, y, z) < radius * radius) {
                volume.set(false, x, y, z);
            }
        }
    }
}

// 3. The 'volume' now contains the final data to be applied to the world.
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never share a single BooleanVoxelSpace instance across multiple threads for writing without implementing an external locking strategy. The non-atomic read-modify-write operations on the underlying integers will cause unpredictable behavior.
-   **Ignoring Bounds:** Do not repeatedly call `set` or `getContent` on coordinates outside the volume. While `set` fails silently, `getContent` will throw an exception. Always use `isInsideSpace` for checks involving external or unpredictable coordinate sources.
-   **Inefficient Iteration:** When filling or copying large volumes, prefer bulk operations like `set(Boolean)` or `deepCopyFrom` over iterating and calling `set(value, x, y, z)` for each voxel individually. The bulk methods are significantly more performant.

## Data Pipeline
BooleanVoxelSpace acts as an in-memory data store and transformation buffer within a larger generation pipeline.

> Flow:
> Generation Algorithm (e.g., Noise Function, Shape Primitive) -> **BooleanVoxelSpace::set** -> (Internal Bit-Packed State) -> **BooleanVoxelSpace::getContent** -> World Chunk Builder -> Final Game World State

