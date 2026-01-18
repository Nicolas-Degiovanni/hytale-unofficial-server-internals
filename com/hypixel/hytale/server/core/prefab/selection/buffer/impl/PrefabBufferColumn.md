---
description: Architectural reference for PrefabBufferColumn
---

# PrefabBufferColumn

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer.impl
**Type:** Transient

## Definition
```java
// Signature
public class PrefabBufferColumn {
```

## Architecture & Concepts
The PrefabBufferColumn is a fundamental, immutable data transfer object (DTO) used within the server's prefab system. It represents a complete, vertical snapshot of a single world column (a slice along the Y-axis at a specific X, Z coordinate) that has been read into a temporary buffer.

This class acts as a data container, holding two primary types of world information:
1.  **Block Data:** A map of `Holder<ChunkStore>` objects, representing the chunks and their block states within that column.
2.  **Entity Data:** An array of `Holder<EntityStore>` objects, representing all entities located within the column's bounds.

Its primary role is to serve as an in-memory data structure that decouples the process of reading world data from the process of writing it. Systems that perform large-scale world modifications, such as pasting a saved prefab or generating a dynamic structure, will first load the source data into a collection of PrefabBufferColumn instances. This buffered data is then used as the source of truth for the subsequent write operation, ensuring atomicity and preventing read/write conflicts on live world storage.

### Lifecycle & Ownership
-   **Creation:** A PrefabBufferColumn is instantiated by higher-level buffer management services, typically a `PrefabBufferReader` or a similar class responsible for scanning a region of the world and serializing it into this in-memory format. It is fully populated via its constructor and is inert upon creation.
-   **Scope:** The object's lifetime is intentionally short and is scoped to a single, high-level world operation. For example, during a "paste" command, a set of these objects will exist only for the duration of that command, from the initial read to the final write.
-   **Destruction:** As a simple POJO with no external resource handles, it is reclaimed by the Java garbage collector once all references to it are dropped. No explicit cleanup is required.

## Internal State & Concurrency
-   **State:** **Immutable**. All fields are declared `final` and are assigned exclusively in the constructor. This guarantees that an instance of PrefabBufferColumn cannot be altered after its creation.

    **Warning:** While the object itself is immutable, it does not perform defensive copies of the collections passed to its constructor. The `entityHolders` array and `blockComponents` map are stored by reference. Modifying these collections externally after instantiation is a severe violation of the class's design contract and will lead to unpredictable behavior.

-   **Thread Safety:** **Conditionally Thread-Safe**. Due to its immutability, a PrefabBufferColumn instance can be safely shared and read by multiple threads without synchronization. The thread safety of the *contained* data (the `EntityStore` and `ChunkStore` objects within the Holders) is determined by those components, not by this container.

## API Surface
The public API is minimal, consisting only of accessors for its internal state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getReaderIndex() | int | O(1) | Returns the original index from the source buffer, used for ordering. |
| getEntityHolders() | Holder<EntityStore>[] | O(1) | Returns the array of entity data holders. May be null. |
| getBlockComponents() | Int2ObjectMap<Holder<ChunkStore>> | O(1) | Returns the map of chunk data holders, keyed by chunk Y-index. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. It is consumed by systems that orchestrate world data manipulation. A typical consumer would retrieve a column from a buffer and iterate through its contents to apply them to the world.

```java
// Conceptual example of a system processing a column
PrefabBufferColumn column = prefabBuffer.getColumn(x, z);

// Apply block data
for (Holder<ChunkStore> chunkHolder : column.getBlockComponents().values()) {
    worldWriter.applyChunk(targetCoords, chunkHolder.get());
}

// Apply entity data
if (column.getEntityHolders() != null) {
    for (Holder<EntityStore> entityHolder : column.getEntityHolders()) {
        worldWriter.spawnEntity(targetCoords, entityHolder.get());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabBufferColumn()`. These objects should only be created by the designated prefab reading and buffering services to ensure data integrity and correct indexing.
-   **State Mutation:** Never attempt to modify the array or map returned by `getEntityHolders()` or `getBlockComponents()`. These are direct references to the internal state, and modifying them breaks the immutability guarantee, which can cause catastrophic and hard-to-debug failures in world generation or editing operations.

## Data Pipeline
PrefabBufferColumn serves as a crucial intermediate step in any data flow involving buffered world modifications.

> Flow:
> World Storage -> Prefab Reader Service -> **PrefabBufferColumn** (In-Memory Buffer) -> Prefab Paster Service -> World Storage

