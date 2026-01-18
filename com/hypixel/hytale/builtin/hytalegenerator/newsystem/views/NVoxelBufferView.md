---
description: Architectural reference for NVoxelBufferView
---

# NVoxelBufferView<T>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.views
**Type:** Transient Facade

## Definition
```java
// Signature
public class NVoxelBufferView<T> implements VoxelSpace<T> {
```

## Architecture & Concepts

The NVoxelBufferView is a high-performance facade that provides a standard VoxelSpace interface over a non-contiguous, chunked data store. It is a fundamental component in the new world generation system, acting as a translation layer between abstract generator logic and the underlying physical buffer layout.

Its primary architectural purpose is to present a simple, contiguous, 3D grid of voxels to a consumer, while the actual data is stored in a collection of discrete NVoxelBuffer instances managed by a NBufferBundle. The view does not own any voxel data; it holds a lightweight reference to a specific region of the NBufferBundle and delegates all read and write operations to the appropriate underlying buffers.

The core responsibility of this class is **coordinate space translation**. World generation algorithms operate in a world-space voxel grid. The NVoxelBufferView uses the GridUtils utility to translate these world-space coordinates into buffer-grid coordinates to locate the correct NVoxelBuffer, and then further translates them into local coordinates within that buffer. This abstraction allows generator code to be written without any knowledge of the underlying chunking or buffering strategy.

## Lifecycle & Ownership

-   **Creation:** NVoxelBufferView instances are intended to be short-lived and are created on-demand by higher-level systems that manage the NBufferBundle. A world generation stage, for example, will request a view for the specific region it intends to modify.

-   **Scope:** The lifecycle of a view is typically bound to a single, discrete operation, such as the execution of a single generator pass. It should be considered a method-scoped object.

-   **Destruction:** The object is garbage collected when it falls out of scope. It holds no native resources and requires no explicit destruction.

**WARNING:** Holding a reference to an NVoxelBufferView for an extended period is a severe anti-pattern. The underlying NBufferBundle may re-organize or invalidate its buffers, leaving the view in a stale and unusable state, which can lead to unpredictable data corruption.

## Internal State & Concurrency

-   **State:** The view itself is effectively stateless beyond its initial configuration (bounds, buffer access). Its fields are initialized in the constructor and never change. However, it provides a mutable interface to the underlying NBufferBundle, which is a highly mutable data structure. The view does not perform any caching of voxel data.

-   **Thread Safety:** **This class is not thread-safe.** All operations on an instance of NVoxelBufferView must be externally synchronized. The underlying NBufferBundle and its accessors are not designed for concurrent access from multiple threads. Accessing the same region of the world via different views on different threads will result in race conditions and undefined behavior. All world generation passes operating on overlapping regions must be serialized.

## API Surface

The public API conforms to the VoxelSpace interface, but many methods are intentionally unsupported. The primary contract is for direct voxel content access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| set(content, position) | boolean | O(1) | Sets the voxel content at the specified world-space position. Delegates to the underlying buffer. |
| getContent(position) | T | O(1) | Retrieves the voxel content from the specified world-space position. |
| copyFrom(source) | void | O(N) | Copies buffer *references* from a source view. N is the number of buffers in the view, not voxels. This is a shallow, high-performance operation. |
| isInsideSpace(position) | boolean | O(1) | Checks if the given coordinates are within the view's bounds. |

**WARNING:** Methods such as `forEach`, `pasteFrom`, and `setOrigin` are not implemented and will throw an UnsupportedOperationException. This view is a specialized tool for direct get and set operations, not a general-purpose voxel manipulation utility.

## Integration Patterns

### Standard Usage

A view should always be acquired from a managing context that controls the lifecycle of the underlying NBufferBundle. The generator then operates on the view without needing to understand the buffer system.

```java
// A generator receives a context for the region it needs to process
public void generateFeature(GeneratorContext context) {
    // Acquire a view for a specific voxel type and region
    NVoxelBufferView<Block> blockView = context.getBlockView();

    // Operate on the view using world-space coordinates
    Vector3i featureCenter = blockView.getCenter();
    blockView.set(Blocks.STONE, featureCenter);
    blockView.set(Blocks.TORCH, featureCenter.x, featureCenter.y + 1, featureCenter.z);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new NVoxelBufferView()`. The `NBufferBundle.Access.View` required by the constructor is an internal component. Always acquire views from the appropriate service or context.

-   **Out-of-Bounds Access:** Accessing coordinates outside the view's pre-defined bounds will trigger an assertion error in development builds and may cause unpredictable memory access in production. Always use `isInsideSpace` for safety checks if necessary.

-   **Storing Views:** Do not store an NVoxelBufferView in a field or cache it for later use. Its validity is tied to the scope of the operation for which it was created. Request a new view for each new, independent operation.

## Data Pipeline

The NVoxelBufferView is a critical component for routing data from generator logic into the correct physical memory location. It acts as a "smart pointer" or "router" within the data pipeline.

> Flow:
> World Generation Stage -> Requests a view for a specific region -> **NVoxelBufferView** is created -> Generator calls `set(content, pos)` -> View translates world coordinates to buffer coordinates -> View delegates write to the correct `NVoxelBuffer` -> Data is now in the `NBufferBundle` for meshing.

