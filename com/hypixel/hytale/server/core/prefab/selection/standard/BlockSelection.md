---
description: Architectural reference for BlockSelection
---

# BlockSelection

**Package:** com.hypixel.hytale.server.core.prefab.selection.standard
**Type:** Transient

## Definition
```java
// Signature
public class BlockSelection implements NetworkSerializable<EditorBlocksChange>, MetricProvider {
```

## Architecture & Concepts

The BlockSelection class is a high-level, in-memory data structure that represents a three-dimensional volume of blocks, fluids, and entities. It serves as the primary container for prefab data and world editing operations, analogous to a clipboard in a content creation tool.

Its core purpose is to decouple a set of world data from the world itself, allowing for complex, transactional modifications. This includes copying, pasting, rotating, flipping, and serializing sections of the game world. It is a foundational component for systems like prefabs, world generation features, and administrative building tools.

A BlockSelection is defined by its contents and a coordinate system. It maintains a local origin (x, y, z) and an anchor point (anchorX, anchorY, anchorZ). All block and entity positions stored within are relative to this local coordinate system. When a BlockSelection is applied to a World, the anchor point determines the precise placement offset.

## Lifecycle & Ownership

-   **Creation:** A BlockSelection is instantiated on-demand. There is no central manager or factory. It is typically created by a system that needs to perform a bulk world operation, such as a command handler processing a world edit command, or a prefab service loading a prefab from storage.
-   **Scope:** The lifetime of a BlockSelection is bound to the operation that creates it. For a simple place command, it may be created, populated, used, and then immediately become eligible for garbage collection. For more interactive features like a player's clipboard, an instance may persist for a longer duration, tied to the player's session state.
-   **Destruction:** Cleanup is managed by the Java garbage collector. As a pure data container with no native resources, it does not require an explicit destruction method.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable and can be memory-intensive. It primarily consists of three collections:
    -   A map of packed long coordinates to BlockHolder records for block data.
    -   A map of packed long coordinates to FluidHolder records for fluid data.
    -   A list of Holder objects for entity data.
    It also stores integer coordinates for its position, anchor, and selection bounds.

-   **Thread Safety:** This class is explicitly designed to be thread-safe.
    -   Access to block and fluid data is guarded by a **ReentrantReadWriteLock** named blocksLock.
    -   Access to entity data is guarded by a separate **ReentrantReadWriteLock** named entitiesLock.
    -   All public methods that read or write to these collections correctly acquire and release the appropriate locks. This design allows multiple threads to safely iterate over the selection (read lock) or for a single thread to safely modify it (write lock).

    **WARNING:** While the class itself is thread-safe, misuse can still lead to deadlocks. Avoid acquiring locks on multiple BlockSelection objects in different orders. Long-running operations within iterator callbacks (like forEachBlock) can cause lock contention and starve writer threads.

## API Surface

The public API provides a comprehensive contract for building, transforming, and applying world edits.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| place(...) | BlockSelection | O(N) | Applies the selection to a world, returning a new BlockSelection containing the *previous* state of the modified blocks. This is critical for undo operations. |
| placeNoReturn(...) | void | O(N) | A more performant version of place that applies the selection without capturing the world's previous state. Use when undo functionality is not required. |
| rotate(...) | BlockSelection | O(N) | Creates and returns a *new*, rotated BlockSelection. The original selection is not modified. |
| flip(...) | BlockSelection | O(N) | Creates and returns a *new*, flipped BlockSelection. The original selection is not modified. |
| addBlockAtWorldPos(...) | void | O(1) | Adds or updates a single block in the selection. Thread-safe. |
| addEntityFromWorld(...) | void | O(1) | Adds an entity to the selection, converting its world coordinates to local coordinates. Thread-safe. |
| forEachBlock(...) | void | O(N) | Iterates over every block in the selection, executing a provided callback. Acquires a read lock for the duration of the iteration. |
| toPacket() | EditorBlocksChange | O(N) | Serializes the selection into a network packet for client-side previews. |

## Integration Patterns

### Standard Usage

The canonical use case involves creating a selection, populating it from a source (like the world or a prefab), transforming it, and finally applying it to a target world location.

```java
// Example: Copying a 10x10x10 area, rotating it, and pasting it elsewhere.

// 1. Define the source area in the world.
World world = ...;
Vector3i sourceMin = new Vector3i(100, 64, 100);
Vector3i sourceMax = new Vector3i(110, 74, 110);

// 2. Create a BlockSelection and copy data from the world.
BlockSelection selection = new BlockSelection();
selection.setAnchor(0, 0, 0); // Set anchor relative to selection corner
selection.setPosition(sourceMin.getX(), sourceMin.getY(), sourceMin.getZ());

for (int x = sourceMin.getX(); x < sourceMax.getX(); x++) {
    for (int y = sourceMin.getY(); y < sourceMax.getY(); y++) {
        for (int z = sourceMin.getZ(); z < sourceMax.getZ(); z++) {
            // This is a simplified copy; the real implementation is more complex.
            selection.addBlockAtWorldPos(x, y, z, world.getBlock(x, y, z), ...);
        }
    }
}

// 3. Perform a transformation. This returns a NEW instance.
BlockSelection rotatedSelection = selection.rotate(Axis.Y, 90);

// 4. Apply the transformed selection to a new position in the world.
Vector3i targetPos = new Vector3i(200, 64, 200);
BlockSelection undoData = rotatedSelection.place(commandSender, world, targetPos, null);
```

### Anti-Patterns (Do NOT do this)

-   **Long-Term Retention:** Do not hold references to large BlockSelection objects indefinitely. They can consume significant heap memory. Their lifecycle should be as short as possible.
-   **Ignoring Returned Instances:** Methods like **rotate**, **flip**, and **relativize** are non-destructive. They return a new, modified instance. Ignoring the return value and continuing to use the original instance is a common logical error.
-   **Nested Locking:** Avoid calling methods on one BlockSelection from within an iterator callback of another BlockSelection. This can easily lead to deadlocks if the locks are acquired in an inconsistent order.
-   **Modification During Iteration:** Do not attempt to modify a BlockSelection from one thread while another thread is iterating over it using **forEachBlock** or **forEachEntity**. While the locks prevent data corruption, this will lead to a **ConcurrentModificationException** or unpredictable behavior.

## Data Pipeline

BlockSelection acts as a central hub for world data, facilitating flows between the game world, network, and storage.

> **Copy Operation Flow:**
> WorldChunk Data -> `BlockSelection.addBlockAtWorldPos` -> **BlockSelection** (Internal Long2ObjectMap)

> **Paste Operation Flow:**
> **BlockSelection** -> `place()` -> `World.getBlockBulkRelative` -> `WorldChunk.setBlock` -> World State Change -> `NotificationHandler.updateChunk` -> Network Packet to Client

> **Client Preview Flow:**
> **BlockSelection** -> `toPacket()` -> `EditorBlocksChange` -> Network Layer -> Client-side Renderer (Ghost Preview)

