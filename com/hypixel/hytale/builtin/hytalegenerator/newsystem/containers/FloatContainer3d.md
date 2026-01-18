---
description: Architectural reference for FloatContainer3d
---

# FloatContainer3d

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.containers
**Type:** Data Structure

## Definition
```java
// Signature
public class FloatContainer3d {
```

## Architecture & Concepts
The FloatContainer3d is a specialized, high-performance data structure designed to store a three-dimensional grid of floating-point values within the world generation system. It serves as a fundamental building block for representing continuous data fields such as density, temperature, or humidity before they are converted into discrete blocks.

Architecturally, its primary role is to act as an in-memory data buffer for a bounded region of the world, often corresponding to a chunk or a larger processing area. The key design principle is the decoupling of the raw data from its spatial coordinates. It achieves this by mapping a logical 3D volume, defined by a Bounds3i, onto a contiguous, one-dimensional float array. This flat memory layout is highly cache-friendly and optimized for the sequential access patterns common in generation algorithms.

The translation from 3D world coordinates to a 1D array index is managed by the GridUtils utility, which employs a YXZ iteration order. This specific memory layout is chosen to optimize for operations that process entire vertical columns of data at once, a frequent requirement in terrain generation.

## Lifecycle & Ownership
-   **Creation:** Instances are created directly via the constructor (`new FloatContainer3d(...)`). They are typically instantiated by a specific world generation pass, such as a noise generator or a terrain shaping module, to hold the output of a calculation for a defined volume.
-   **Scope:** The lifetime of a FloatContainer3d is transient and scoped to a single, discrete generation task. It exists only as long as needed to compute, store, and pass its data to the next stage in the generation pipeline.
-   **Destruction:** The object is managed by the Java garbage collector. Once the generation stage that created it completes and all references are released, the container and its underlying float array become eligible for garbage collection. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** The internal state is highly **mutable**. The core `data` array is intended to be populated and modified throughout its lifecycle. The coordinate system, represented by `bounds_voxelGrid`, is also mutable via the `moveMinTo` method. The initial bounds and size, however, are fixed upon creation.

-   **Thread Safety:** This class is **not thread-safe** and provides no internal synchronization. It is designed for use within a single worker thread. Concurrent writes from multiple threads will result in data corruption and race conditions. Concurrent reads are safe, but any write operation concurrent with a read or another write constitutes a severe programming error.

    **WARNING:** All external access must be synchronized if an instance is ever shared between threads, though this is strongly discouraged by design.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(position_voxelGrid) | float | O(1) | Retrieves the value at a given 3D position. Performs a full bounds check and returns a default value if the position is outside the container's volume. |
| set(position_voxelGrid, value) | void | O(1) | Sets the value at a given 3D position. **WARNING:** This method uses an assertion for bounds checking, which is disabled in production builds. Calling this with an out-of-bounds position in a production environment will cause an unchecked exception or memory corruption. |
| moveMinTo(min_voxelGrid) | void | O(1) | Translates the container's coordinate system without altering the underlying data array. This is a low-cost operation to change the logical position of the data volume in world space. |
| getBounds_voxelGrid() | Bounds3i | O(1) | Returns a direct reference to the internal bounds object. **WARNING:** Modifying the returned object externally will break the container's internal consistency. |

## Integration Patterns

### Standard Usage
A FloatContainer3d is typically used as a temporary data store within a multi-stage generation process. A generator populates it, and a consumer reads from it.

```java
// 1. Define the volume for the generation task
Bounds3i generationBounds = new Bounds3i(0, 0, 0, 15, 255, 15);
float defaultValue = -1.0f;

// 2. Create a container to hold the output
FloatContainer3d densityMap = new FloatContainer3d(generationBounds, defaultValue);

// 3. A generator populates the container
NoiseGenerator.fillWithDensity(densityMap);

// 4. A subsequent process consumes the data
Voxelizer.convertDensityToBlocks(densityMap);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never allow two threads to call `set` on the same instance simultaneously. This is the most common and severe misuse of this class.
-   **Unchecked Writes:** Do not call `set` with a position that *might* be outside the container's bounds. The lack of a production-time bounds check is a deliberate performance optimization that shifts responsibility to the caller. Always validate coordinates before writing.
-   **External Bounds Mutation:** Do not modify the Bounds3i object returned by `getBounds_voxelGrid`. This bypasses the class's internal logic and will lead to incorrect index calculations. Use the `moveMinTo` method for all coordinate system adjustments.

## Data Pipeline
The FloatContainer3d acts as a stateful buffer between computational stages in the world generation pipeline. It does not define a pipeline itself but is a critical component within one.

> Flow:
> Noise Generation Algorithm -> **FloatContainer3d** (stores raw density values) -> Biome Modification Stage -> **FloatContainer3d** (stores modified density) -> Voxelization Stage -> BlockContainer3d

