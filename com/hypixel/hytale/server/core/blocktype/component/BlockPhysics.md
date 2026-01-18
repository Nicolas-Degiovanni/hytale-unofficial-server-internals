---
description: Architectural reference for BlockPhysics
---

# BlockPhysics

**Package:** com.hypixel.hytale.server.core.blocktype.component
**Type:** Component / Data Container

## Definition
```java
// Signature
public class BlockPhysics implements Component<ChunkStore> {
```

## Architecture & Concepts

The **BlockPhysics** component is a server-side data container responsible for storing low-level physical properties for every block within a single chunk section. It is not a system that executes physics simulations; rather, it provides the underlying data that physics and game logic systems consume.

Architecturally, it serves as a memory-optimized data layer attached directly to a **ChunkStore**. Its primary design goal is to store a 4-bit integer (a nibble) for each of the 32,768 possible block locations in a chunk section. This nibble represents a "support" value, a general-purpose piece of data used by other systems to determine block stability, behavior, or type.

Key architectural features include:

*   **Lazy Allocation:** The internal `supportData` byte array is `null` by default. It is only allocated when the first non-zero support value is written to a block in the chunk section. This is a critical memory optimization, preventing the allocation of 16KB for every single chunk section in the world, most of which may not contain any special physics data.
*   **Automatic Deallocation:** If all support values within the component are set back to zero, the internal `supportData` array is deallocated and set back to `null`, reclaiming its memory. This is managed by the `nonZeroCount` state variable.
*   **Packed Data Structure:** Data is stored as packed nibbles within a byte array. The **BitUtil** utility is used to perform the bitwise operations required to read and write a 4-bit value without disturbing the adjacent value in the same byte.
*   **Specialized Values:** The component defines constants for specific states. For example, a support value of 15 (**IS_DECO_VALUE**) is reserved to flag a block as a "decoration," which may exempt it from certain physics calculations or structural integrity checks.

This component is fundamental for server-side logic that needs to associate persistent, per-block metadata without the overhead of creating a full block entity.

### Lifecycle & Ownership
-   **Creation:** A **BlockPhysics** instance is never created directly with `new`. It is instantiated on-demand by the component system when a system calls `store.ensureAndGetComponent(section, getComponentType())` on a **ChunkStore** reference. This typically occurs the first time a block in that chunk section needs its support value modified.
-   **Scope:** The lifecycle of a **BlockPhysics** instance is strictly tied to its parent **ChunkStore**. It persists as long as the chunk section is loaded in server memory.
-   **Destruction:** The component is marked for garbage collection when its parent **ChunkStore** is unloaded from memory. As noted above, its internal byte array may be garbage collected earlier if its data is cleared, even while the component instance itself remains.

## Internal State & Concurrency
-   **State:** The component's state is mutable and consists of two fields:
    1.  `supportData`: A `byte[]` of size 16384, lazily allocated. This array holds the packed 4-bit support values for 32,768 blocks.
    2.  `nonZeroCount`: An integer tracking the number of non-zero nibbles in the `supportData` array. It is used to determine when to deallocate the array.

-   **Thread Safety:** This component is **thread-safe**. All read and write operations on the internal state are protected by a **StampedLock**.
    -   Write operations (`set`) acquire a full `writeLock`, ensuring atomic updates to the `supportData` array and `nonZeroCount`.
    -   Read operations (`get`) acquire a `readLock`, allowing for concurrent reads but blocking writes.

    **WARNING:** The use of a lock implies that frequent, contentious access to the same **BlockPhysics** component from multiple threads could become a performance bottleneck. Systems should be designed to minimize lock contention where possible.

## API Surface

The primary interaction points are the static helper methods, which abstract away the component's lifecycle management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| set(index, support) | boolean | O(1) | Sets the 4-bit support value for a block at a given linear index. Manages internal state and lock. |
| get(index) | int | O(1) | Retrieves the 4-bit support value for a block. Manages internal read lock. |
| isDeco(index) | boolean | O(1) | Convenience method to check if a block's support value is 15. |
| setSupportValue(store, section, x, y, z, value) | static void | O(1) | **Primary API.** Ensures the component exists and sets the support value for a block at world coordinates. |
| markDeco(store, section, x, y, z) | static void | O(1) | **Primary API.** Ensures the component exists and sets the support value to 15 (decoration). |
| clear(store, section, x, y, z) | static void | O(1) | **Primary API.** Sets a block's support value to 0. Does not create the component if it is missing. |

## Integration Patterns

### Standard Usage

Interaction with **BlockPhysics** should always be performed through its static helper methods, providing a component accessor (**Store** or **Holder**) and a reference to the target chunk section. This pattern correctly handles the on-demand creation and retrieval of the component.

```java
// How a developer should normally use this
// 'world' is a ComponentAccessor<ChunkStore>
// 'sectionRef' is a Ref<ChunkStore> for the target chunk section

// Mark a block as a decoration
BlockPhysics.markDeco(world, sectionRef, 10, 64, 20);

// Set a custom support value
BlockPhysics.setSupportValue(world, sectionRef, 11, 64, 20, 8);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BlockPhysics()`. The resulting object will not be managed by the engine's component system, will not be associated with any chunk, and will not be persisted.
-   **Manual Component Management:** Avoid manually calling `store.getComponent`, checking for null, and then calling `store.ensureAndGetComponent`. The static helper methods provide a cleaner and safer abstraction for this exact workflow.
-   **Ignoring Threading:** While the component is thread-safe, do not assume that a sequence of calls is atomic. For example, a `get` followed by a `set` is not an atomic read-modify-write operation. If such atomicity is required, it must be handled by a higher-level system lock.

## Data Pipeline

**BlockPhysics** acts as a data source and sink, primarily during chunk serialization and deserialization. It does not actively process data streams.

> **Serialization Flow (Chunk Unload):**
> Game State -> **BlockPhysics** internal `supportData` -> `BlockPhysics.serialize()` -> `CODEC` -> `byte[]` -> ChunkStore -> Disk / Network

> **Deserialization Flow (Chunk Load):**
> Disk / Network -> `byte[]` -> ChunkStore -> `CODEC` -> `BlockPhysics.deserialize()` -> **BlockPhysics** internal `supportData` -> Game State

