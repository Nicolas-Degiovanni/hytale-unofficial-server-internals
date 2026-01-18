---
description: Architectural reference for HalfByteSectionPalette
---

# HalfByteSectionPalette

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.palette
**Type:** Transient

## Definition
```java
// Signature
public class HalfByteSectionPalette extends AbstractByteSectionPalette {
```

## Architecture & Concepts

The HalfByteSectionPalette is a memory-optimized data structure responsible for storing block data for a single 16x16x16 chunk section. It is a concrete implementation of the Palette design pattern, specifically engineered for scenarios with low block-type diversity.

This class is a critical component of the server's world memory management strategy. It can represent up to 16 unique block types within a 4096-block volume. Instead of storing a full integer ID for each block, it stores a 4-bit (a "nibble" or "half-byte") index. This index maps to the full block ID via an internal lookup table. This results in a significant memory reduction from 16KB per section (using direct integer IDs) to approximately 2KB.

HalfByteSectionPalette exists within a state machine of palette implementations. It is the second level of optimization, sitting between the EmptySectionPalette (for sections with one block type) and the ByteSectionPalette (for sections with 17-256 block types). The system dynamically **promotes** this palette to a ByteSectionPalette if the number of unique blocks exceeds 16, or **demotes** it to an EmptySectionPalette if all blocks become uniform. This dynamic resizing ensures optimal memory usage across the entire world.

## Lifecycle & Ownership

-   **Creation:** A HalfByteSectionPalette is never instantiated directly by application logic. It is created under two conditions:
    1.  By a parent ChunkSection during world generation or modification when the number of unique block types is between 2 and 16.
    2.  As the result of a **demotion** from a ByteSectionPalette when its unique block count drops to 16 or fewer. This is handled by the static factory method `fromBytePalette`.

-   **Scope:** The lifecycle of a HalfByteSectionPalette is strictly bound to its parent ChunkSection. It persists in memory as long as the chunk is loaded on the server.

-   **Destruction:** The object is eligible for garbage collection immediately after its parent ChunkSection is unloaded or when it is replaced by a promoted or demoted palette instance. There are no manual cleanup procedures.

## Internal State & Concurrency

-   **State:** This object is highly mutable. Its primary state consists of:
    -   A `byte[] blocks` array storing the packed 4-bit palette indices for all 4096 block positions.
    -   Bidirectional maps (`internalToExternal`, `externalToInternal`) for resolving palette indices to global block IDs.
    -   A `Byte2ShortMap internalIdCount` tracking the usage count of each palette entry, which is critical for determining when demotion is possible.

-   **Thread Safety:** **This class is not thread-safe.** All operations that modify the palette's state, including setting blocks and promotion/demotion, must be executed on the primary world thread for the corresponding dimension. Unsynchronized access from multiple threads will lead to race conditions, resulting in severe world data corruption, inconsistent client-side rendering, and server instability.

## API Surface

The primary interaction with this class is through the ISectionPalette interface, which is not detailed here. The methods below define its unique behavior within the palette system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| promote() | ByteSectionPalette | O(N) | Creates and returns a new ByteSectionPalette, migrating all existing block data. The caller is responsible for replacing its reference to this palette with the new one. |
| demote() | ISectionPalette | O(1) | Returns the singleton instance of EmptySectionPalette. This is only valid if the palette contains a single block type. |
| fromBytePalette(section) | HalfByteSectionPalette | O(N) | Static factory to create a new HalfByteSectionPalette by demoting a ByteSectionPalette. Throws IllegalStateException if the source palette has more than 16 unique blocks. |
| getPaletteType() | PaletteType | O(1) | Returns the enum constant PaletteType.HalfByte, used for network serialization. |

## Integration Patterns

### Standard Usage

Developers should never interact with this class directly. All block modifications occur through higher-level world APIs, which delegate to the ChunkSection. The ChunkSection manages the palette's lifecycle, including promotion.

```java
// Conceptual example of the system promoting a palette
// This logic resides within a ChunkSection or similar manager.

ISectionPalette currentPalette = this.palette;
int newBlockExternalId = 42; // e.g. "hytale:stone"

// This call might add a 17th unique block type
currentPalette.setBlock(x, y, z, newBlockExternalId);

// The palette implementation itself checks if it needs to be promoted
if (currentPalette.shouldPromote()) {
    // The promotion creates a NEW instance with the data migrated.
    // The old palette must be replaced.
    this.palette = currentPalette.promote();
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new HalfByteSectionPalette()`. Palettes must be created by the ChunkSection or via the defined promotion/demotion state transitions to ensure world integrity.
-   **State Mutation After Promotion:** Do not retain and use a reference to a HalfByteSectionPalette after calling `promote()` on it. The promotion method returns a *new* object that replaces the old one. Continued use of the old reference will lead to lost block updates.
-   **Multi-threaded Access:** Never read from or write to a palette from any thread other than the world's main tick thread. This will cause catastrophic data corruption.

## Data Pipeline

The HalfByteSectionPalette acts as a compression and decompression layer for block data within a ChunkSection.

> **Block Write Flow:**
> World API `setBlock(pos, externalId)` -> ChunkSection finds or creates internal ID -> **HalfByteSectionPalette** checks capacity -> If capacity exceeded, `promote()` is called and the old palette is discarded -> `set0` is called with internal ID -> `BitUtil.setNibble` writes 4-bit ID to the internal byte array.

> **Network Serialization Flow:**
> Server prepares chunk for client -> ChunkSection requests palette data -> **HalfByteSectionPalette** provides its type (`HalfByte`), its ID mapping, and the raw `blocks` byte array -> Data is written to the network packet for client-side reconstruction.

