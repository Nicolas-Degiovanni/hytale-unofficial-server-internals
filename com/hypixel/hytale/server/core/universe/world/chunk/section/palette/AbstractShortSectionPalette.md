---
description: Architectural reference for AbstractShortSectionPalette
---

# AbstractShortSectionPalette

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.palette
**Type:** Foundational Component / Data Structure

## Definition
```java
// Signature
public abstract class AbstractShortSectionPalette implements ISectionPalette {
```

## Architecture & Concepts

The AbstractShortSectionPalette is a base implementation for the **Palette** compression pattern, a critical memory optimization for voxel-based worlds. In a naive implementation, storing a full 32-bit block ID for every voxel in a 32x32x32 chunk section would consume 128KB. The palette pattern reduces this by storing only the *unique* block IDs present in a section (the "palette") and then using a smaller, local index to reference them.

This abstract class provides the core logic for managing this two-way mapping. It handles the translation between global, engine-wide block IDs (external IDs) and the compressed, section-local palette indices (internal IDs). It is the bridge between the logical world representation and the physical, memory-optimized storage format.

A key architectural feature is the concept of **promotion**. This class has a finite capacity for unique block types, determined by the bit-size of its concrete implementation. When a `set` operation attempts to add a new unique block ID that exceeds this capacity, it returns the status `REQUIRES_PROMOTE`. This signals the calling system (typically a ChunkSection) that it must replace this palette with a more capable one, such as a palette with a larger bit-size or a direct-mapping storage that forgoes palettes entirely.

Subclasses are expected to implement the low-level bit-packing logic for reading and writing internal IDs from the underlying `short[]` data array via the `get0` and `set0` methods.

### Lifecycle & Ownership
- **Creation:** This is an abstract class and cannot be instantiated directly. Concrete implementations are created by a factory or manager when a `ChunkSection` is first loaded or generated. The palette is initialized either from a deserialized data stream or as an empty container for a new section.
- **Scope:** The lifecycle of a palette is strictly bound to its parent `ChunkSection`. It persists in memory for as long as the section is active on the server.
- **Destruction:** The object is eligible for garbage collection when its parent `ChunkSection` is unloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** This component is highly **mutable**. Its primary purpose is to manage the state of a chunk section's block data. Operations like `set` and `deserialize` directly modify the internal maps, reference counts, and the core `blocks` data array. The state includes:
    - A bidirectional map between global block IDs and local palette indices.
    - A reference count for each unique block type within the section.
    - The raw, compressed block data for the entire section.
- **Thread Safety:** This class is **not thread-safe**. The internal `fastutil` collections are not synchronized. Concurrent modification of the palette from multiple threads will lead to state corruption, including incorrect reference counts, broken mappings, and unpredictable block data.

    **WARNING:** All interactions with a palette instance must be synchronized externally, typically by ensuring all world modifications occur on a single, designated world-processing thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(index) | int | O(1) | Retrieves the global block ID at a given voxel index. |
| set(index, id) | ISectionPalette.SetResult | O(1) avg | Sets the global block ID at an index. Manages palette entries and reference counts. Returns a result indicating if the palette requires promotion. |
| count() | int | O(1) | Returns the number of unique block types currently in the palette. |
| contains(id) | boolean | O(1) avg | Checks if a global block ID is present in the palette. |
| serialize(serializer, buf) | void | O(N) | Writes the palette mappings and block data to a ByteBuf for persistent storage. |
| deserialize(deserializer, buf, version) | void | O(N) | Reads and reconstructs the palette state from a ByteBuf. |

## Integration Patterns

### Standard Usage
The palette is almost never used in isolation. It is owned and managed by a `ChunkSection`, which orchestrates block updates and handles palette promotion.

```java
// Hypothetical usage within a ChunkSection class

// Get the palette for this section
ISectionPalette palette = this.getPalette();

// Set a block at local coordinates (x, y, z)
int index = toIndex(x, y, z);
ISectionPalette.SetResult result = palette.set(index, newGlobalBlockId);

// CRITICAL: Handle palette promotion
if (result == ISectionPalette.SetResult.REQUIRES_PROMOTE) {
    ISectionPalette newPalette = promotePalette(palette);
    newPalette.set(index, newGlobalBlockId);
    this.setPalette(newPalette);
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Promotion:** Failing to check for and handle the `REQUIRES_PROMOTE` result from the `set` method is a critical error. It will prevent new block types from being added to a full section, effectively causing data loss.
- **Multi-threaded Access:** Accessing a single palette instance from multiple threads without external locking will corrupt its internal state, leading to server instability and world corruption.
- **State Caching:** Do not cache the result of `palette.count()` or other state-dependent methods across ticks. The palette's state can change with any block update.

## Data Pipeline

The AbstractShortSectionPalette acts as a translation and compression layer within the world data pipeline.

**Block Write Operation:**
> Flow:
> World Update Logic -> `ChunkSection.setBlock(pos, id)` -> **`AbstractShortSectionPalette.set(index, id)`** -> Internal Maps & Counts Updated -> Compressed `blocks` array modified

**Block Read Operation:**
> Flow:
> World Query Logic -> `ChunkSection.getBlock(pos)` -> **`AbstractShortSectionPalette.get(index)`** -> Read from compressed `blocks` array -> Lookup in `internalToExternal` map -> Return global block ID

**Chunk Serialization:**
> Flow:
> Server Save Process -> `Chunk.serialize()` -> `ChunkSection.serialize()` -> **`AbstractShortSectionPalette.serialize(buf)`** -> ByteBuf -> Network or Disk Storage

