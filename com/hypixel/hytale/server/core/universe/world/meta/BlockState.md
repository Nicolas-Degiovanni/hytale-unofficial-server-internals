---
description: Architectural reference for BlockState
---

# BlockState

**Package:** com.hypixel.hytale.server.core.universe.world.meta
**Type:** Component

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public abstract class BlockState extends Object implements Component<ChunkStore> {
```

## Architecture & Concepts

The BlockState component is the server-side data model for blocks that possess complex state beyond their fundamental type. While a simple stone block only needs a type identifier, a block such as a chest, furnace, or sign requires an associated data structure to manage its inventory, fuel level, or text content. BlockState serves as the abstract base class for these data structures.

**WARNING:** This entire class hierarchy is deprecated and scheduled for removal. New development should avoid integrating with this system.

Architecturally, BlockState is a core component within the server's Entity-Component-System (ECS) framework, specifically for world data managed by the ChunkStore. It acts as the bridge between the low-level chunk data grid and high-level game logic systems. When a system needs to interact with a stateful block, it queries the ECS for the corresponding BlockState component associated with that block's position.

Serialization is a key responsibility. BlockState and its subclasses are managed by a polymorphic `CodecMapCodec`, allowing the system to serialize a heterogeneous collection of different block states into a BSON document for world storage and deserialize them back into concrete Java objects upon loading.

## Lifecycle & Ownership

The lifecycle of a BlockState instance is strictly bound to its parent WorldChunk. It does not exist independently of the chunk it occupies.

-   **Creation:** BlockState instances are never created directly via a constructor. They are instantiated by one of two mechanisms:
    1.  **Deserialization:** During chunk loading, the static `BlockState.load` method is invoked by the chunk loading pipeline. It uses the master `CODEC` to decode a BSON document into a concrete BlockState subclass.
    2.  **In-Game Placement:** When a player or system places a new stateful block, the `BlockStateModule.createBlockState` factory method is called to create a new, default instance. This is often triggered by the deprecated `ensureState` helper.

-   **Scope:** An instance persists in memory only as long as its parent WorldChunk is loaded. If the chunk is unloaded, the BlockState instance is destroyed.

-   **Destruction:** The `onUnload` method is the primary hook for cleanup logic. This is invoked by the WorldChunk just before it is removed from memory, allowing the BlockState to release any resources or finalize its state.

## Internal State & Concurrency

-   **State:** BlockState is fundamentally mutable. It maintains references to its parent WorldChunk, its chunk-local position, and its unique handle within the ECS (the `Ref<ChunkStore>`). Subclasses add further mutable state, such as inventories or timers. The `initialized` flag tracks whether the component has been fully linked to the world after creation.

-   **Thread Safety:** This class is **not thread-safe**. All interactions with a BlockState instance must be performed on the main server world thread. While chunk loading and saving may occur on background threads, the BlockState objects themselves are managed and manipulated synchronously within the primary game loop. The `AtomicBoolean` for the initialized flag is a narrow-scoped protection for the initialization sequence, not a guarantee of general thread safety.

    **WARNING:** Accessing or modifying a BlockState from an asynchronous task without proper synchronization with the main world thread will lead to race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialize(BlockType) | boolean | O(1) | Finalizes initialization after creation. Must be called before the object is used by game systems. |
| onUnload() | void | O(1) | Lifecycle callback invoked by the parent WorldChunk immediately before it is unloaded. |
| getPosition() | Vector3i | O(1) | Returns the chunk-local position of the block. Returns a clone to prevent mutation. |
| getBlockPosition() | Vector3i | O(1) | Calculates and returns the absolute world coordinates of the block. |
| getChunk() | WorldChunk | O(1) | Returns a reference to the parent chunk containing this block state. |
| markNeedsSave() | void | O(1) | Flags the parent chunk as dirty, ensuring this BlockState's data is persisted to disk. |
| saveToDocument() | BsonDocument | O(N) | Serializes the object's state into a BSON document for storage. N depends on subclass complexity. |
| toHolder() | Holder | O(C) | Packages this component and all its sibling components into a generic Holder object for ECS operations. C is the number of components on the entity. |
| load(doc, chunk, pos) | BlockState | O(N) | Static factory method to deserialize a BSON document into a fully initialized BlockState instance. |

## Integration Patterns

### Standard Usage

Game logic systems should always retrieve BlockState instances through the world or chunk APIs. Direct references should not be held across game ticks, as the underlying chunk may unload.

```java
// A system needs to interact with a block at a specific world position.
World world = ...;
Vector3i worldPos = new Vector3i(100, 64, -50);

// Retrieve the chunk and the state.
WorldChunk chunk = world.getChunkAtBlock(worldPos.x, worldPos.z);
if (chunk != null) {
    BlockState state = chunk.getState(worldPos.x & 31, worldPos.y, worldPos.z & 31);

    // Perform type-specific logic
    if (state instanceof ChestBlockState) {
        ChestBlockState chest = (ChestBlockState) state;
        chest.getInventory().addItem(...);
        
        // IMPORTANT: Mark the chunk for saving after modification.
        state.markNeedsSave();
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ConcreteBlockState()`. The object will be uninitialized, lack a world reference, and will not be managed by the ECS, causing severe runtime errors. Always use the provided factory methods like `BlockStateModule.createBlockState`.

-   **State Caching:** Do not store a direct reference to a BlockState instance in a long-lived system. The chunk can unload at any time, invalidating the reference and leading to `NullPointerException` or use-after-free bugs. Re-fetch the BlockState from the chunk when needed or use the `Ref<ChunkStore>` handle.

-   **Forgetting to Save:** Modifying a BlockState without calling `markNeedsSave()` on it (or `chunk.markNeedsSaving()`) will result in the changes being lost upon server restart or chunk unload.

## Data Pipeline

The BlockState component is a critical link in the world persistence data pipeline. Its flow is bidirectional, moving between in-memory representation and on-disk storage.

> **Load Flow:**
> Disk Storage (Chunk File) -> BSON Document -> `WorldChunk` Deserializer -> **`BlockState.load()`** -> In-Memory `BlockState` Object -> Game Logic Systems

> **Save Flow:**
> Game Logic Systems -> Modify In-Memory `BlockState` Object -> **`state.markNeedsSave()`** -> `WorldChunk` Serializer -> **`BlockState.saveToDocument()`** -> BSON Document -> Disk Storage (Chunk File)

