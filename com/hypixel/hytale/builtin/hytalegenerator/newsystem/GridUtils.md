---
description: Architectural reference for GridUtils
---

# GridUtils

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem
**Type:** Utility

## Definition
```java
// Signature
public class GridUtils {
```

## Architecture & Concepts

The GridUtils class is a stateless, static utility that serves as the canonical library for coordinate space transformations within the world generation system. It provides a centralized and authoritative set of functions for converting positions and bounding boxes between the three primary grid systems:

*   **Voxel Grid:** The highest-resolution grid, representing the absolute position of a single block in the world. This is the fundamental unit of world space.
*   **Buffer Grid:** A coarse grid where each cell corresponds to a single NVoxelBuffer. An NVoxelBuffer is a fixed-size container of voxel data. Based on the bit-shift operations (a shift of 3 bits), a buffer represents an 8x8x8 volume of voxels.
*   **Chunk Grid:** The highest-level grid used for organizing the world into manageable sections for loading, saving, and processing. A chunk is composed of a 4x40x4 grid of NVoxelBuffers, resulting in a 32x320x32 voxel volume.

This class is critical for generator pipeline stages, enabling algorithms to operate in the most efficient coordinate space. For example, broad-phase feature placement might operate in the Chunk Grid, while detailed terrain carving operates in the Voxel Grid. GridUtils provides the mathematical bridge between these stages, ensuring precision and consistency.

## Lifecycle & Ownership

*   **Creation:** As a static utility class, GridUtils is never instantiated. Its methods are accessed directly via the class name (e.g., GridUtils.toVoxelGrid_fromBufferGrid).
*   **Scope:** The class is loaded by the JVM's classloader and its methods are available for the entire application lifetime.
*   **Destruction:** The class is unloaded when the application terminates and its classloader is garbage collected. There is no manual cleanup required.

## Internal State & Concurrency

*   **State:** GridUtils is entirely **stateless**. It contains no member variables and all methods are static. The behavior of its functions depends exclusively on the arguments provided at the time of the call.
*   **Thread Safety:** The class is inherently **thread-safe**. Since it holds no state, concurrent calls from multiple world generation threads cannot interfere with one another.

    **WARNING:** Several methods modify their input arguments in-place (e.g., toVoxelGrid_fromBufferGrid(Bounds3i)). While the GridUtils class itself is thread-safe, passing a shared, mutable object like a Vector3i or Bounds3i to these methods from multiple threads without external synchronization will lead to race conditions and unpredictable behavior. Callers are responsible for ensuring thread safety of the arguments they provide.

## API Surface

The API consists of static methods for coordinate conversion and bounding box creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toBufferGrid_fromVoxelGrid(Vector3i) | void | O(1) | Converts a Voxel Grid position to a Buffer Grid position, modifying the input vector in-place. |
| toVoxelGrid_fromBufferGrid(Bounds3i) | void | O(1) | Converts a Buffer Grid bounding box to a Voxel Grid bounding box, modifying the input bounds in-place. |
| createChunkBounds_voxelGrid(int, int) | Bounds3i | O(1) | Creates and returns a new bounding box representing the full volume of a world chunk in Voxel Grid coordinates. |
| createBufferBoundsInclusive_fromVoxelBounds(Bounds3i) | Bounds3i | O(1) | Calculates the Buffer Grid bounds required to fully contain a given Voxel Grid bounds. Returns a new object. |
| toIndexFromPositionYXZ(Vector3i, Bounds3i) | int | O(1) | Flattens a 3D position within a bounding box into a 1D array index. Asserts that the position is within the bounds. |

## Integration Patterns

### Standard Usage

GridUtils should be used whenever a system needs to translate coordinates or regions between the different grid systems managed by the world generator.

```java
// Example: Determine the voxel-space bounds of a chunk and then
// find the buffer-space bounds that fully contain it.

// 1. Get the chunk's bounds in the finest-grained coordinate system (voxels).
Bounds3i chunkInVoxelGrid = GridUtils.createChunkBounds_voxelGrid(10, -5);

// 2. Convert these voxel bounds to the coarser buffer grid for processing.
// This is useful for iterating over all buffers that the chunk occupies.
Bounds3i chunkInBufferGrid = GridUtils.createChunkBounds_bufferGrid(10, -5);

// 3. A voxel position can be converted to its local position inside a buffer.
Vector3i worldVoxelPos = new Vector3i(164, 88, 135);
GridUtils.toVoxelGridInsideBuffer_fromWorldGrid(worldVoxelPos);
// worldVoxelPos is now (4, 0, 7), the local coordinate within its parent buffer.
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Do not create an instance with `new GridUtils()`. This is a pure static utility class. Instantiating it serves no purpose and adds minor overhead. All methods must be accessed statically.
*   **Ignoring In-Place Modification:** Many methods modify the vector or bounds object passed as an argument. This is done for performance to avoid object allocation. Failure to account for this can lead to subtle bugs where an object's state is unexpectedly changed.

    ```java
    // BAD: The original voxelBounds object is mutated and its original value is lost.
    Bounds3i voxelBounds = new Bounds3i(min, max);
    GridUtils.toBufferGrid_fromVoxelGridOverlap(voxelBounds);
    // The variable voxelBounds now holds buffer grid coordinates.

    // GOOD: Clone the object if the original value is needed later.
    Bounds3i voxelBounds = new Bounds3i(min, max);
    Bounds3i bufferBounds = voxelBounds.clone();
    GridUtils.toBufferGrid_fromVoxelGridOverlap(bufferBounds);
    // voxelBounds remains unchanged, bufferBounds holds the result.
    ```

## Data Pipeline

GridUtils does not process a stream of data itself. Instead, it acts as a set of transformation functions applied at various stages within a larger data pipeline, such as world generation. It enables the translation of a request from one coordinate system to another as it flows through the system.

> **World Generation Flow Example:**
>
> Chunk Generation Request (Chunk Coords) -> **GridUtils.createChunkBounds_voxelGrid** -> Voxel Bounding Box -> Terrain Generation Stage (operates on voxels) -> **GridUtils.createBufferBoundsInclusive_fromVoxelBounds** -> Buffer Bounding Box -> Post-Processing Stage (operates on buffers) -> Final Chunk Data Structure

