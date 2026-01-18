---
description: Architectural reference for AbstractByteSectionPalette
---

# AbstractByteSectionPalette

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.palette
**Type:** Transient Data Structure

## Definition
```java
// Signature
public abstract class AbstractByteSectionPalette implements ISectionPalette {
```

## Architecture & Concepts

The AbstractByteSectionPalette provides the foundational implementation for a memory-optimized block storage strategy used within world chunk sections. It employs the **Palette** design pattern to significantly reduce the memory footprint of a chunk section, which is a 32x32x32 volume of blocks.

Instead of storing the full 32-bit global block identifier (the *external ID*) for each of the 32,768 positions, this system maintains a small, localized mapping. Each unique block type present in the section is assigned a compact *internal ID*, represented here as a byte. The main block data is then stored as a large array of these internal byte IDs.

This class manages the bidirectional mapping between external and internal IDs:
*   **External ID (int):** The global, unique identifier for a block type across the entire game (e.g., 1 for Stone, 2 for Dirt).
*   **Internal ID (byte):** A local, per-palette identifier valid only within this chunk section. It serves as an index into the palette.

The core responsibility of this class is to manage the lifecycle of palette entries. It uses reference counting to track how many blocks of each type exist. When the count for a block type drops to zero, its entry is removed from the palette, freeing up an internal ID for reuse.

A critical feature of this architecture is the concept of **promotion**. Because the internal ID is a byte, a single palette can only contain a maximum of 256 unique block types. If an operation attempts to add a 257th unique block, the `set` method will return `REQUIRES_PROMOTE`. This signals to the owning system (typically a ChunkSection) that this palette is full and must be upgraded to a more capable implementation, such as one using a short or int for its internal IDs.

## Lifecycle & Ownership

-   **Creation:** An AbstractByteSectionPalette is instantiated by its owning `ChunkSection` when the section is first generated or loaded from storage. It can be created in an empty state (containing only Air) or be populated from a deserialized data stream.
-   **Scope:** The lifecycle of a palette instance is strictly tied to its parent `ChunkSection`. It persists in memory as long as the chunk section is active on the server.
-   **Destruction:** The object is eligible for garbage collection when its parent `ChunkSection` is unloaded from memory. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency

-   **State:** This object is highly **mutable**. Any call to the `set` method can modify the internal-to-external ID maps, the reference counts, and the underlying `blocks` byte array. It is a stateful container for a specific region of world data.

-   **Thread Safety:** **This class is not thread-safe.** All internal data structures, including the fastutil maps and the `blocks` array, are accessed without any synchronization mechanisms.

    **WARNING:** Concurrent modification will lead to severe data corruption, including palette desynchronization, incorrect block counts, and server instability. All interactions with a palette instance must be performed on the main server thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int index) | int | O(1) | Retrieves the external block ID at the given linearized index. |
| set(int index, int id) | ISectionPalette.SetResult | O(1) avg | Sets the block at the index to the specified external ID. Manages all internal reference counting and palette entry creation. Returns a result indicating if the palette requires promotion. |
| count() | int | O(1) | Returns the number of unique block types currently in the palette. |
| serialize(serializer, buf) | void | O(N) | Writes the palette's mapping and block data to a ByteBuf for persistence. N is the number of unique block types. |
| deserialize(deserializer, buf, version) | void | O(N) | Populates the palette from a ByteBuf. Clears all prior state. |

## Integration Patterns

### Standard Usage

The AbstractByteSectionPalette is not meant to be used directly by high-level game logic. It is an internal component managed by a `ChunkSection`. The `ChunkSection` acts as a facade, handling the promotion logic when required.

```java
// Conceptual example within a ChunkSection class
ISectionPalette currentPalette = new ByteNibbleSectionPalette(); // A concrete implementation

public void setBlock(int index, int blockId) {
    ISectionPalette.SetResult result = this.currentPalette.set(index, blockId);

    if (result == ISectionPalette.SetResult.REQUIRES_PROMOTE) {
        // The palette is full and must be upgraded
        ISectionPalette newPalette = createPromotedPalette(this.currentPalette);
        this.currentPalette = newPalette;
        
        // Retry the operation on the new, larger palette
        this.currentPalette.set(index, blockId); 
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Ignoring Promotion:** Failing to check for and handle the `REQUIRES_PROMOTE` result from the `set` method will prevent new block types from being added once the palette is full, leading to data loss.
-   **Concurrent Access:** Accessing a palette instance from multiple threads will corrupt its internal state. All world modifications must be synchronized through the main server tick.
-   **Direct State Modification:** Directly modifying the public `blocks` array or other internal collections from outside the class will break the reference counting and mapping invariants, causing unpredictable behavior.

## Data Pipeline

The primary data flow involves translating between game-level block operations and the compressed, in-memory representation.

> **Flow: Set Block Operation**
>
> Server Game Logic -> `ChunkSection.setBlock(index, externalId)` -> **`AbstractByteSectionPalette.set(index, externalId)`** -> Update Internal Maps & Reference Counts -> Write new internal ID to `blocks` array

