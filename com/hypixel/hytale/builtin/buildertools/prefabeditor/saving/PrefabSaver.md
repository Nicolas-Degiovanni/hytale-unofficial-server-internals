---
description: Architectural reference for PrefabSaver
---

# PrefabSaver

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.saving
**Type:** Utility

## Definition
```java
// Signature
public class PrefabSaver {
```

## Architecture & Concepts
The PrefabSaver is a static utility class that serves as the primary orchestrator for serializing a three-dimensional region of a game world into a persistent prefab file. It acts as a high-level facade over the complex subsystems responsible for world data, including the ChunkStore, EntityStore, and the final serialization layer, PrefabStore.

Its core responsibility is to translate a spatial query (defined by a bounding box) into a portable, in-memory representation called a BlockSelection. This process involves several critical steps:
1.  **Chunk Preloading:** Identifying and synchronously loading all world chunks that intersect with the target selection area. This is a performance-critical step to ensure all necessary data is available in memory.
2.  **Data Extraction:** Iterating through every block coordinate within the selection and copying block, fluid, and block entity state data.
3.  **Entity Aggregation:** Querying for and copying all entities within the selection bounds.
4.  **Serialization Hand-off:** Passing the fully populated BlockSelection object to the PrefabStore for final processing and disk I/O.

The entire operation is designed to be executed asynchronously off the main server thread, leveraging the World's task execution system via CompletableFuture.

## Lifecycle & Ownership
As a static utility class, PrefabSaver has no instance lifecycle.
-   **Creation:** It is never instantiated. All its methods are static.
-   **Scope:** Its methods are available for the entire lifetime of the server process after the class is loaded by the JVM.
-   **Destruction:** Not applicable. The class is unloaded when the server shuts down.

## Internal State & Concurrency
-   **State:** PrefabSaver is entirely stateless. It holds no instance or static fields that represent mutable data. All state is passed into its methods as arguments and returned as new objects, making its operations functionally pure from an internal state perspective.

-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, the operations it performs are computationally expensive and involve significant I/O. The primary entry point, savePrefab, returns a CompletableFuture and submits its work to the World object, which acts as an Executor.

    **WARNING:** The chunk preloading mechanism contains a blocking wait loop (`while (!future.isDone())`). While this occurs within an asynchronous task, it will monopolize the worker thread it is running on, preventing it from processing other world tasks until all required chunks are loaded from disk. This can lead to performance bottlenecks if chunk loading is slow or the selection area is very large.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| savePrefab(...) | CompletableFuture<Boolean> | O(W\*H\*D) | Asynchronously copies and saves a world region to a prefab file. W, H, D are the dimensions of the selection. Returns a future that completes with true on success, false on failure. |

## Integration Patterns

### Standard Usage
The class should be invoked from server-side logic, typically a command handler, in response to a user action. The caller provides the world, selection coordinates, and save path, then handles the result asynchronously.

```java
// Standard invocation from a server command or plugin
CompletableFuture<Boolean> saveFuture = PrefabSaver.savePrefab(
    sender,
    world,
    savePath,
    anchor,
    min,
    max,
    pastePos,
    originalAnchor,
    settings
);

saveFuture.thenAccept(success -> {
    if (success) {
        sender.sendMessage(Message.translation("...success..."));
    } else {
        sender.sendMessage(Message.translation("...failure..."));
    }
});
```

### Anti-Patterns (Do NOT do this)
-   **Blocking on the Future:** Calling `saveFuture.join()` or `saveFuture.get()` on the main server thread will freeze the server until the entire save operation is complete, which can be seconds or minutes for large prefabs.

-   **Excessively Large Selections:** Attempting to save an extremely large area can lead to `OutOfMemoryError` as the entire BlockSelection is constructed in memory before serialization. It will also cause severe performance degradation due to the blocking chunk load and exhaustive iteration.

-   **Ignoring Special Blocks:** The logic explicitly filters or re-maps special editor blocks like Editor_Block and Editor_Empty. Developers extending this system must be aware that not all blocks in the world are copied verbatim.

## Data Pipeline
The flow of data through the PrefabSaver is a multi-stage extraction and aggregation process.

> Flow:
> 1.  **Input:** A set of Vector3i coordinates defining a bounding box, anchor, and paste position within a World.
> 2.  **Chunk Identification:** The bounding box is used to calculate a set of unique chunk indices that cover the area.
> 3.  **Synchronous Chunk Load:** The system iterates through the required chunk indices and issues asynchronous load requests to the ChunkStore. It then enters a busy-wait loop, consuming the world task queue until all futures complete.
> 4.  **Voxel Iteration:** The code performs a nested loop over every X, Y, and Z coordinate within the bounding box.
> 5.  **Data Extraction:** For each coordinate, it accesses the corresponding loaded ChunkColumn and BlockSection to retrieve block ID, block state, fluid ID, and fluid level. This data is copied into an in-memory **BlockSelection** object.
> 6.  **Entity Extraction:** A spatial query is made against the EntityStore to find all entities within the bounding box. Each entity is copied and added to the **BlockSelection**.
> 7.  **Serialization:** The completed **BlockSelection** object is passed to the PrefabStore, which handles the final conversion to a binary format and writes it to the specified file path on disk.

