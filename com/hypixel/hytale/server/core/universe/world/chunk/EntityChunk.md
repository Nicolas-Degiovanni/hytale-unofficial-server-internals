---
description: Architectural reference for EntityChunk
---

# EntityChunk

**Package:** com.hypixel.hytale.server.core.universe.world.chunk
**Type:** Component (Data Container)

## Definition
```java
// Signature
public class EntityChunk implements Component<ChunkStore> {
```

## Architecture & Concepts
The EntityChunk is a fundamental data component within Hytale's Entity Component System (ECS) architecture. It is not a manager or a service; it is a passive data container attached to a **ChunkStore** entity. Its sole responsibility is to maintain a manifest of entities that reside within the spatial boundaries of a specific world chunk.

This class embodies the critical distinction between an entity's persisted state and its live, in-memory state. It achieves this by managing two distinct collections:

1.  **Entity Holders (unloaded):** A list of Holder objects. A Holder is a serialized, inert representation of an entity and its components. These are entities that belong to the chunk but are not currently active in the game world because the chunk is not loaded. This collection is the source of truth for persistence.

2.  **Entity References (loaded):** A set of Ref objects. A Ref is a lightweight, direct pointer to a live, ticking entity within the global EntityStore. These represent the active entities currently simulated within a loaded chunk.

The logic that transitions entities between these two states is handled externally by the **EntityChunkLoadingSystem**. This separation of data (EntityChunk) from logic (the System) is a core tenet of ECS design, ensuring that EntityChunk remains a simple and predictable data structure.

## Lifecycle & Ownership
The lifecycle of an EntityChunk is inextricably tied to its parent ChunkStore entity. It does not and cannot exist independently.

-   **Creation:** An EntityChunk instance is created when its parent ChunkStore entity is instantiated. This occurs either during world generation or, more commonly, when a chunk is deserialized from disk. The static CODEC field defines the serialization contract for loading and saving its state.

-   **Scope:** The component persists for the entire lifetime of the in-memory ChunkStore entity. It is a session-scoped object relative to the chunk's presence in memory.

-   **Destruction:** The instance is marked for garbage collection when the parent ChunkStore entity is destroyed, typically when a chunk is unloaded from memory and has been fully persisted to disk. The EntityChunkLoadingSystem guarantees that all live entity references are converted back to holders before destruction.

## Internal State & Concurrency
-   **State:** EntityChunk is a highly mutable component. Its internal collections are constantly modified as entities enter or leave the chunk's boundaries, or as the chunk itself is loaded and unloaded from memory. It caches the state of unloaded entities via its Holder list and uses a boolean *needsSaving* flag to implement a dirty-checking mechanism, optimizing disk I/O operations.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. All state modifications are expected to be performed synchronously within the server's tick loop, typically by ECS Systems. The internal collections (ObjectArrayList, HashSet) are not synchronized. Unmanaged multi-threaded access will lead to state corruption, data loss, and server instability.

## API Surface
The public API is designed for state management by trusted core systems, not for general gameplay logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addEntityHolder(holder) | void | O(1) | Adds a serialized entity holder. Marks the chunk for saving. |
| addEntityReference(ref) | void | O(1) | Adds a live entity reference. Marks the chunk for saving. |
| removeEntityReference(ref) | void | O(1) | Removes a live entity reference. Marks the chunk for saving. |
| takeEntityHolders() | Holder[] | O(N) | Drains and returns all entity holders. Resets the internal list. |
| takeEntityReferences() | Ref[] | O(N) | Drains and returns all entity references. Resets the internal set. |
| markNeedsSaving() | void | O(1) | Manually flags the chunk as modified, ensuring it will be saved. |
| consumeNeedsSaving() | boolean | O(1) | Returns the current save flag and resets it to false. |

## Integration Patterns

### Standard Usage
Direct interaction with EntityChunk is rare. Core engine systems, primarily EntityChunkLoadingSystem, are its main consumers. These systems acquire the component from a ChunkStore entity to orchestrate the loading and unloading of entities.

```java
// Executed within an ECS System on the main server thread
void processChunkLoad(Ref<ChunkStore> chunkRef, Store<ChunkStore> chunkStore) {
    EntityChunk entityChunk = chunkStore.getComponent(chunkRef, EntityChunk.getComponentType());
    if (entityChunk == null) {
        return; // Chunk has no entity data
    }

    // The system takes ownership of the serialized holders to activate them
    Holder<EntityStore>[] holdersToLoad = entityChunk.takeEntityHolders();

    if (holdersToLoad != null) {
        // ... logic to add entities to the world from these holders
        // and populate the entityChunk with live references.
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new EntityChunk()`. It is a component managed exclusively by the ChunkStore's ECS lifecycle. Manually created instances will not be tracked or persisted.

-   **State Mutation without Flagging:** Avoid methods like `storeEntityHolder` or `loadEntityReference` which modify state without setting the `needsSaving` flag. These are for internal use by loading systems only and will result in data loss if used improperly.

-   **External Iteration:** Do not retain references to the collections returned by `getEntityHolders` or `getEntityReferences`. These are unmodifiable views and their underlying collections can be changed at any time by engine systems.

-   **Asynchronous Modification:** Never modify an EntityChunk from another thread. All interactions must be scheduled as commands on the main server CommandBuffer to prevent race conditions.

## Data Pipeline
EntityChunk is a critical waypoint in the entity persistence and activation data pipeline.

**Flow 1: Chunk Loading (Disk to Live)**
> Chunk File on Disk -> Deserialization via **EntityChunk.CODEC** -> `EntityChunk` populated with `entityHolders` -> **EntityChunkLoadingSystem** reads `entityHolders` -> Entities added to global `EntityStore` -> `EntityChunk` populated with live `entityReferences`

**Flow 2: Chunk Unloading (Live to Disk)**
> **EntityChunkLoadingSystem** reads `entityReferences` from `EntityChunk` -> Entities removed from global `EntityStore` (returns `Holders`) -> `EntityChunk` populated with `entityHolders` -> Serialization via **EntityChunk.CODEC** -> Chunk File on Disk

