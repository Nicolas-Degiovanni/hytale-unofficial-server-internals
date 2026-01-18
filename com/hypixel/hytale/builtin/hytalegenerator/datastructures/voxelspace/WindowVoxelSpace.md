---
description: Architectural reference for WindowVoxelSpace
---

# WindowVoxelSpace<T>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.voxelspace
**Type:** Stateful Wrapper

## Definition
```java
// Signature
public class WindowVoxelSpace<T> implements VoxelSpace<T> {
```

## Architecture & Concepts
The WindowVoxelSpace is an implementation of the **Decorator** design pattern. It provides a constrained, virtual "view" into a sub-region of another, larger VoxelSpace without copying any underlying voxel data. It acts as a lightweight, stateful proxy that forwards all read and write operations to the wrapped VoxelSpace instance.

Its primary architectural purpose is to simplify algorithms that operate on bounded regions of the world. Instead of passing boundary coordinates (minX, minY, minZ, maxX, maxY, maxZ) through complex call stacks, a system can create a WindowVoxelSpace for the target region and pass this single object to subsequent logic. The receiving algorithm can then treat the window as a complete, self-contained VoxelSpace, with all its operations (iteration, bounds checking) automatically constrained to the window's dimensions.

Key characteristics:
*   **No Data Duplication:** It holds a reference to the original VoxelSpace, not a copy of its data.
*   **Read/Write Proxy:** Modifications made through the WindowVoxelSpace directly alter the state of the wrapped VoxelSpace.
*   **Dynamic Resizing:** The window's boundaries can be redefined at any time via the setWindow method, allowing a single instance to be reused for operations on different sub-regions.

This class is fundamental to world generation features like structure placement, biome decoration, and cave carving, where procedural logic must be applied to a specific, localized volume within a larger chunk or world segment.

## Lifecycle & Ownership
- **Creation:** Instantiated directly and on-demand by a client system. It is not a managed service or a singleton. Typically, a world generation process will create a WindowVoxelSpace, operate on it, and then discard it.
- **Scope:** The lifecycle of a WindowVoxelSpace is intended to be short and bound to a specific operation. It is a transient object.
- **Destruction:** The object is eligible for garbage collection as soon as it goes out of scope.

**WARNING:** The WindowVoxelSpace does not own the VoxelSpace it wraps. The caller is responsible for ensuring that the wrapped VoxelSpace remains valid for the entire lifetime of the window. Allowing the wrapped VoxelSpace to be destroyed while a window still references it will result in undefined behavior and likely memory access violations.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its internal state consists of a reference to the wrapped VoxelSpace and two VoxelCoordinate objects representing the window's minimum and maximum corners. This state is modified by the setWindow method.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. All read and write operations are passed directly to the wrapped VoxelSpace. Therefore, the thread safety of a WindowVoxelSpace is entirely dependent on the thread safety of the object it wraps.
    - Concurrent calls to setWindow while other threads are accessing voxel data through the same instance will lead to severe race conditions.
    - All access to a single WindowVoxelSpace instance must be externally synchronized if used across multiple threads.

## API Surface
The public contract is defined by the VoxelSpace interface, with one critical addition for configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setWindow(minX, ..., maxZ) | WindowVoxelSpace<T> | O(1) | **Primary configuration method.** Defines the sub-region this instance will represent. Throws IllegalArgumentException if bounds are invalid. |
| getContent(x, y, z) | T | O(1) | Retrieves voxel data. Throws IllegalArgumentException if coordinates are outside the *window's* bounds. Delegates to the wrapped space. |
| set(content, x, y, z) | boolean | O(1) | Sets voxel data. Returns false if coordinates are outside the *window's* bounds. Delegates to the wrapped space. |
| getWrappedSchematic() | VoxelSpace<T> | O(1) | Returns a direct reference to the underlying VoxelSpace. |
| setOrigin(x, y, z) | void | O(1) | **Always throws UnsupportedOperationException.** A window cannot change the fundamental origin of the underlying data structure. |
| forEach(action) | void | O(N) | Iterates over every voxel *within the window's bounds*, where N is the volume of the window (sizeX * sizeY * sizeZ). |

## Integration Patterns

### Standard Usage
The correct pattern is to create a window, define its bounds, pass it to a processing system, and then discard it.

```java
// 1. Obtain a reference to a large-scale VoxelSpace (e.g., a chunk)
VoxelSpace<Block> chunkSpace = world.getChunkVoxelSpace(0, 0, 0);

// 2. Create a window and configure its sub-region
WindowVoxelSpace<Block> structureArea = new WindowVoxelSpace<>(chunkSpace);
structureArea.setWindow(10, 64, 10, 26, 80, 26); // Define a 16x16x16 area

// 3. Pass the window to a system that operates on a VoxelSpace
StructurePlacer placer = new StructurePlacer();
placer.buildHouse(structureArea); // The placer only sees a 16x16x16 space
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of WindowVoxelSpace in long-lived data structures. They are intended for transient, operational use. Storing them can prevent the underlying wrapped VoxelSpace from being garbage collected.
- **Concurrent Boundary Changes:** Do not call setWindow from one thread while another thread is iterating or accessing data using the same instance. This will corrupt the state of the operation.

    ```java
    // BAD: Race condition
    WindowVoxelSpace<Block> sharedWindow = new WindowVoxelSpace<>(chunk);
    
    // Thread 1
    new Thread(() -> sharedWindow.forEach(...)).start();
    
    // Thread 2
    new Thread(() -> sharedWindow.setWindow(0,0,0,8,8,8)).start(); // This will cause undefined behavior in Thread 1
    ```
- **Ignoring Thrown Exceptions:** Do not attempt to catch and ignore the UnsupportedOperationException from setOrigin. This indicates a fundamental misunderstanding of the class's purpose. A window is a view, not an independent entity with its own origin.

## Data Pipeline
A call to a WindowVoxelSpace method is a direct, synchronous proxy. The flow for a write operation is simple and introduces minimal overhead.

> Flow:
> External System Call -> `WindowVoxelSpace.set(data, x, y, z)` -> Internal Bounds Check (is x,y,z inside window?) -> `wrappedVoxelSpace.set(data, x, y, z)` -> Underlying Data Store (e.g., Chunk's block array)

