---
description: Architectural reference for ByteSectionPalette
---

# ByteSectionPalette

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.palette
**Type:** Transient State Object

## Definition
```java
// Signature
public class ByteSectionPalette extends AbstractByteSectionPalette {
```

## Architecture & Concepts

The ByteSectionPalette is a memory-optimization component responsible for storing block data within a single 32x32x32 chunk section. It implements a palette compression scheme, acting as the intermediate tier between the highly compressed HalfByteSectionPalette and the more flexible ShortSectionPalette.

This class is employed when a chunk section contains between 15 and 256 unique block types. Instead of storing the full 32-bit global ID for each of the 32,768 blocks in the section, it maps each unique global ID to a local 8-bit (byte) identifier. This local ID is then stored in the main block data array, significantly reducing the memory footprint of the chunk section from approximately 128KB (uncompressed) to just over 32KB plus a small mapping table.

The core responsibility of this class is to manage the bidirectional mapping between global and local IDs and to provide mechanisms for transitioning to a different palette implementation when the number of unique blocks changes.

### Lifecycle & Ownership
-   **Creation:** A ByteSectionPalette is almost never created directly. It comes into existence through one of two state transitions managed by a parent object, typically a ChunkSection:
    1.  **Promotion:** When a HalfByteSectionPalette exceeds its capacity of 14 unique blocks, it is promoted into a new ByteSectionPalette instance via the static `fromHalfBytePalette` factory method.
    2.  **Demotion:** When a ShortSectionPalette's unique block count falls to 256 or fewer, it can be demoted into a new ByteSectionPalette via the `fromShortPalette` factory method.

-   **Scope:** The lifetime of a ByteSectionPalette instance is ephemeral and strictly tied to the state of its parent ChunkSection. It persists only as long as the section's unique block count remains within the 15-256 range.

-   **Destruction:** The object is eligible for garbage collection as soon as its owning ChunkSection replaces it with a different palette implementation (either through promotion to a ShortSectionPalette or demotion to a HalfByteSectionPalette). There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This class is highly mutable. Its primary state consists of the `blocks` byte array and several mapping tables inherited from its parent: `externalToInternal`, `internalToExternal`, `internalIdSet`, and `internalIdCount`. These collections are modified whenever a block is set or removed, or during promotion and demotion operations.

-   **Thread Safety:** **This class is not thread-safe.** All internal data structures, including the fastutil maps and the core `blocks` array, are accessed without synchronization. Any concurrent modification will lead to an inconsistent state, resulting in severe data corruption within the world. All interactions with a ByteSectionPalette instance must be externally synchronized, typically by the single world thread responsible for managing the corresponding chunk.

## API Surface

The primary contract of this class revolves around its ability to transition to other palette types. Direct block manipulation is handled by the `get` and `set` methods of its parent, AbstractByteSectionPalette.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| promote() | ShortSectionPalette | O(N) | Creates a new ShortSectionPalette, migrating all existing block data. Used when unique blocks exceed 256. |
| demote() | HalfByteSectionPalette | O(N) | Creates a new HalfByteSectionPalette, migrating all existing block data. Used when unique blocks fall to 14 or less. |
| shouldDemote() | boolean | O(1) | Returns true if the number of unique blocks is low enough to warrant demotion to a more memory-efficient palette. |
| fromHalfBytePalette(palette) | ByteSectionPalette | O(N) | **Static Factory.** Creates a new ByteSectionPalette by promoting a HalfByteSectionPalette. |
| fromShortPalette(palette) | ByteSectionPalette | O(N) | **Static Factory.** Creates a new ByteSectionPalette by demoting a ShortSectionPalette. Throws IllegalStateException if the source palette has more than 256 unique blocks. |

*Complexity N refers to the number of blocks in the section (32768).*

## Integration Patterns

### Standard Usage

A developer should never interact with this class directly. The parent ChunkSection is responsible for managing its lifecycle. The standard pattern involves checking the palette's state after a block update and triggering a promotion or demotion to replace the entire palette object.

```java
// Hypothetical usage within a ChunkSection class
private AbstractByteSectionPalette palette;

public void setBlock(int x, int y, int z, int globalBlockId) {
    // This call might increase the unique block count
    int newCount = this.palette.set(x, y, z, globalBlockId);

    // Check for promotion
    if (newCount > ByteSectionPalette.MAX_SIZE && this.palette instanceof ByteSectionPalette) {
        // WARNING: This replaces the entire palette object
        this.palette = ((ByteSectionPalette) this.palette).promote();
    }

    // Check for demotion (less common on set)
    if (this.palette.shouldDemote()) {
        // ... demotion logic ...
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ByteSectionPalette()`. This creates an empty, uninitialized palette. Palettes must only be created by transitioning from an existing, valid palette to ensure data integrity is maintained.
-   **Stateful Caching:** Do not hold a reference to a ByteSectionPalette instance. The owning ChunkSection can replace it at any time with a promoted or demoted version. Always re-request the palette from the ChunkSection.
-   **Concurrent Access:** Never access a palette from multiple threads. All world modifications must be queued and processed by a single, synchronized world thread to prevent race conditions and data corruption.

## Data Pipeline

The ByteSectionPalette acts as a translation layer for block data within a chunk section.

> **Write Flow:**
> Global Block ID -> `ChunkSection.setBlock` -> `AbstractByteSectionPalette.set` -> **ByteSectionPalette** (maps global ID to internal byte ID) -> `blocks` array write

> **Read Flow:**
> `ChunkSection.getBlock` -> `AbstractByteSectionPalette.get` -> **ByteSectionPalette** (reads internal byte ID from `blocks` array) -> Map lookup (byte ID to global ID) -> Return Global Block ID

> **Network Serialization Flow:**
> `ChunkSection` -> Get `PaletteType.Byte` -> Serialize ID Mappings -> Serialize `blocks` byte array -> Network Packet

