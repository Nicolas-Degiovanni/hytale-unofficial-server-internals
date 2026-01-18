---
description: Architectural reference for ISectionPalette
---

# ISectionPalette

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.palette
**Type:** Contract / Strategy

## Definition
```java
// Signature
public interface ISectionPalette {
    // ... methods
}
```

## Architecture & Concepts
The ISectionPalette interface is a critical memory optimization component within the world data structure. It embodies the **Flyweight** and **Strategy** design patterns to drastically reduce the memory footprint of a ChunkSection, which represents a 16x16x16 volume of blocks.

Instead of storing a full global block ID (e.g., a 16-bit integer) for each of the 4096 positions within a section, the system stores a small, local index. This index points to an entry in a "palette" of unique block IDs present only within that specific section.

For example, a section composed entirely of Air and Stone only requires a 1-bit index per block, plus a small palette array containing the global IDs for Air and Stone. This is a significant saving over storing two 16-bit integers 4096 times.

The interface defines the contract for all palette implementations. The engine dynamically selects the most efficient concrete implementation (e.g., HalfByteSectionPalette, ByteSectionPalette) based on the number of unique block types a section contains. This selection and the ability to transition between strategies is the core function of this system.

## Lifecycle & Ownership
- **Creation:** Concrete ISectionPalette instances are not created directly. They are instantiated by a parent ChunkSection during world generation or when a chunk is loaded from disk or network. The static factory method, ISectionPalette.from, is the canonical entry point for creating the most memory-efficient palette for a given dataset.

- **Scope:** The lifecycle of a palette is strictly bound to its parent ChunkSection. It exists as long as the ChunkSection is held in memory.

- **Mutation & Transition:** The most important lifecycle events are promotion and demotion.
    - **Promotion:** When a new block type is added to a section and the current palette is full (e.g., adding a 17th unique block to a HalfByteSectionPalette which can only hold 16), the `set` method returns **REQUIRES_PROMOTE**. The owning ChunkSection is then responsible for calling `promote()`, which returns a new, larger palette instance (e.g., a ByteSectionPalette) with all existing data migrated. The ChunkSection must then replace its reference to the old palette with the new one.
    - **Demotion:** Conversely, if blocks are removed and the number of unique types drops below a threshold, `shouldDemote()` will return true. The ChunkSection can then call `demote()` to create a new, smaller, more memory-efficient palette.

- **Destruction:** The object is eligible for garbage collection when its owning ChunkSection is unloaded and no longer referenced. There is no explicit cleanup method.

## Internal State & Concurrency
- **State:** All concrete implementations of ISectionPalette are highly **mutable**. Methods like `set` directly modify the internal arrays that store the block indices and the palette of unique IDs. They are stateful by design.

- **Thread Safety:** **This interface and its implementations are NOT thread-safe.** All operations that modify a palette, such as `set`, `promote`, or `demote`, must be executed exclusively on the main world thread for the corresponding dimension. Unsynchronized access from other threads will lead to data corruption, race conditions, and server crashes. Read operations like `get` are only safe if no concurrent writes are occurring.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| set(pos, id) | SetResult | O(1) | Sets the block ID at a given position. Returns a result indicating the outcome, critically including REQUIRES_PROMOTE if the palette is full. |
| get(pos) | int | O(1) | Retrieves the global block ID at a given position. |
| promote() | ISectionPalette | O(N) | Creates and returns a new, larger palette instance, migrating all existing data. The caller is responsible for replacing the old palette. |
| demote() | ISectionPalette | O(N) | Creates and returns a new, smaller palette instance, migrating all existing data. |
| serializeForPacket(buf) | void | O(N) | Writes the palette type, unique ID list, and packed block data to a Netty ByteBuf for network transmission. |
| from(data, unique, count) | ISectionPalette | O(1) | **Static Factory.** The primary construction method. Analyzes the number of unique IDs and returns the most memory-efficient concrete implementation. |

## Integration Patterns

### Standard Usage
Interaction with the palette is almost always mediated by the parent ChunkSection. The section's block modification methods handle the promotion/demotion lifecycle internally.

```java
// A ChunkSection would perform this logic internally
// when its setBlock method is called.

// 1. Attempt to set the block
ISectionPalette.SetResult result = currentPalette.set(position, blockId);

// 2. Handle the case where the palette is full
if (result == ISectionPalette.SetResult.REQUIRES_PROMOTE) {
    // 3. Promote to a larger palette
    ISectionPalette newPalette = currentPalette.promote();

    // 4. Replace the old palette with the new one
    this.currentPalette = newPalette;

    // 5. Retry the operation on the new, larger palette
    newPalette.set(position, blockId);
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Promotion:** Failing to check for and handle the REQUIRES_PROMOTE result from the `set` method. This will lead to a failure to place the new block and potential data desynchronization.
- **Concurrent Modification:** Accessing a single palette instance from multiple threads without external locking. This will corrupt chunk data.
- **State Mismatch:** Continuing to use a reference to an old palette after it has been promoted or demoted. The owning ChunkSection must update its reference to the new instance returned by `promote()` or `demote()`.

## Data Pipeline
The ISectionPalette is a key component in the world serialization and networking pipeline. It transforms the in-memory representation of a 16x16x16 block volume into a compact binary format.

> **Serialization Flow (Server -> Client):**
> ChunkSection Data -> **ISectionPalette.serializeForPacket()** -> Compacted Block Indices & ID Map -> Netty ByteBuf -> Network Packet

> **Deserialization Flow (Client <- Server):**
> Network Packet -> Netty ByteBuf -> **ISectionPalette.deserialize()** -> Reconstructed ISectionPalette Instance -> Populated ChunkSection

