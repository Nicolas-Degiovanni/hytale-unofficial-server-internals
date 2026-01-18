---
description: Architectural reference for ShortSectionPalette
---

# ShortSectionPalette

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.palette
**Type:** Transient

## Definition
```java
// Signature
public class ShortSectionPalette extends AbstractShortSectionPalette {
```

## Architecture & Concepts
The ShortSectionPalette is a high-capacity data structure responsible for compressing block data within a 16x16x16 ChunkSection. It implements the palette compression pattern, which optimizes memory and network bandwidth by replacing global block IDs with smaller, locally-scoped indices.

This class represents the second tier in the palette promotion system. It is used when the number of unique block types in a section exceeds the capacity of a ByteSectionPalette (251 unique types) but is less than or equal to 65,536. It acts as a state within a finite state machine for chunk data representation:

*   **SingleBlockPalette:** A section with only one block type.
*   **ByteSectionPalette:** A section with 2 to 251 unique block types.
*   **ShortSectionPalette:** A section with 252 to 65,536 unique block types.

If a section requires more than 65,536 unique block types, the palette system is abandoned in favor of a direct storage model using global IDs. The ShortSectionPalette is therefore the most complex and highest-capacity palette implementation before this transition. Its primary role is to manage the bidirectional mapping between 32-bit global block IDs and 16-bit local palette indices.

### Lifecycle & Ownership
- **Creation:** A ShortSectionPalette is never created directly during gameplay logic. It comes into existence exclusively through two mechanisms:
    1.  **Promotion:** When a ByteSectionPalette's unique block count exceeds its limit, the owning ChunkSection will trigger a promotion, creating a new ShortSectionPalette instance via the static factory method `fromBytePalette`.
    2.  **Deserialization:** When a chunk is loaded from disk or received over the network and its metadata indicates a short-based palette, this class is instantiated and populated with the serialized data.

- **Scope:** The lifetime of a ShortSectionPalette is strictly bound to its parent ChunkSection. It is an ephemeral data container that is frequently replaced.

- **Destruction:** The object is eligible for garbage collection under two conditions:
    1.  When its parent ChunkSection is unloaded from memory.
    2.  When the number of unique blocks within the section drops below the demotion threshold (251), causing the ChunkSection to replace this instance with a new, more efficient ByteSectionPalette.

## Internal State & Concurrency
- **State:** The ShortSectionPalette is highly mutable. Its core state consists of:
    - A `short[] blocks` array of size 32768, storing the local palette index for each block position.
    - Several `fastutil` primitive maps that manage the mapping between global integer IDs and internal short indices.
    - A count of unique block types, which is used to determine if the palette should be demoted.

- **Thread Safety:** **This class is not thread-safe.** It is a raw data structure designed for performance within a single-threaded context. All access and modification must be externally synchronized, typically by the main world-update thread that owns the corresponding chunk. Unsynchronized concurrent access will lead to severe data corruption, including invalid block mappings and incorrect block counts.

## API Surface
The public API is minimal, focusing on state transitions (promotion/demotion) and type identification. The primary block manipulation logic is inherited from AbstractShortSectionPalette.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPaletteType() | PaletteType | O(1) | Returns the enum constant `PaletteType.Short` for network serialization. |
| shouldDemote() | boolean | O(1) | Checks if the unique block count is at or below the demotion threshold (251). |
| demote() | ByteSectionPalette | O(N) | Creates a new, more memory-efficient ByteSectionPalette from this instance's data. N is the number of blocks (32768). |
| promote() | ISectionPalette | O(1) | Throws UnsupportedOperationException. This is the highest-tier palette and cannot be promoted further. |
| fromBytePalette(section) | ShortSectionPalette | O(M + N) | **Static Factory.** Creates a new ShortSectionPalette from a ByteSectionPalette. M is the size of the old palette, N is the block count. |

## Integration Patterns

### Standard Usage
The ShortSectionPalette is managed entirely by its owning ChunkSection. A developer should never interact with it directly. The typical interaction is indirect, where a change to a ChunkSection might trigger a palette promotion.

```java
// Conceptual example within a ChunkSection class
// WARNING: This is a simplified representation of the engine's internal logic.

ISectionPalette currentPalette = this.palette;

// This setBlock call might add a new unique block type
currentPalette.setBlock(x, y, z, newGlobalBlockId);

// After modification, the ChunkSection checks if the palette needs to change state
if (currentPalette instanceof ByteSectionPalette && currentPalette.shouldPromote()) {
    // The engine promotes the palette, replacing the old instance
    this.palette = ShortSectionPalette.fromBytePalette((ByteSectionPalette) currentPalette);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new ShortSectionPalette()`. The default constructor creates an empty, invalid state. Palettes must be created via promotion from a lower-tier palette or deserialization.
- **Concurrent Modification:** Never access or modify a palette instance from any thread other than the one that owns the world region. This will break the internal state of the maps and counts.
- **Stale References:** Do not retain a reference to a palette instance after its parent ChunkSection has promoted or demoted it. The retained instance will be stale and out of sync with the canonical world state.

## Data Pipeline
The ShortSectionPalette is a critical component in the block data serialization and modification pipeline.

> **Block Modification Flow:**
> World::setBlock() -> Chunk::setBlock() -> ChunkSection::setBlock() -> ISectionPalette::set() -> **ShortSectionPalette** updates internal maps and block array -> Potentially triggers a demotion on next tick.

> **Serialization Flow (Saving to Disk / Sending over Network):**
> Chunk Save Operation -> ChunkSection Serialization -> **ShortSectionPalette** provides its internal maps and block array -> Serializer writes palette type, unique ID map, and block data array to the output stream.

