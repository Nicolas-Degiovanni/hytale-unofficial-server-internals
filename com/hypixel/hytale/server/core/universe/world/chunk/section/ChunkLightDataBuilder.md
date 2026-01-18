---
description: Architectural reference for ChunkLightDataBuilder
---

# ChunkLightDataBuilder

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section
**Type:** Transient

## Definition
```java
// Signature
public class ChunkLightDataBuilder extends ChunkLightData {
```

## Architecture & Concepts

The ChunkLightDataBuilder is a high-performance, mutable builder for creating immutable ChunkLightData instances. Its primary role is to provide an efficient in-memory representation of lighting data for a 32x32x32 chunk section, which can be modified before being compacted into a final, network-optimized format.

The core of this class is a **sparse octree** data structure, stored within a Netty ByteBuf. This structure is critical for performance and memory efficiency. Lighting data within a game world is often highly compressible; large volumes of air share the same sky light, and deep underground caverns share the same block light (zero). An octree exploits this by representing large, uniform volumes of light with a single node, rather than storing data for every single block.

Each node in the octree represents a cubic volume and is encoded as a 17-byte segment:
*   **1-byte mask:** A bitmask where each bit corresponds to one of the 8 child octants. If a bit is set, the corresponding child is a pointer to another 17-byte node (a sub-division). If the bit is clear, the child is a leaf, and its value is stored directly.
*   **8x 2-byte shorts:** An array of 8 values. Depending on the mask, each short is either a uniform light value for an entire octant or an integer pointer (offset) to the start of a child node's 17-byte segment within the same buffer.

This builder manages a potentially sparse and fragmented buffer during modification. The final `build` method compacts this working representation into a dense, sequential buffer suitable for serialization and network transfer.

### Lifecycle & Ownership
- **Creation:** A ChunkLightDataBuilder is instantiated under two conditions:
    1.  To create entirely new lighting data, typically during initial chunk generation.
    2.  To modify existing lighting data. In this case, an existing ChunkLightData object is passed to the constructor, which "decompresses" the compact octree into the mutable, sparse format required for editing. This process involves rebuilding the segment map by traversing the source octree.

- **Scope:** This object is **transient and short-lived**. It is designed to exist only for the duration of a single, atomic lighting update operation. It should be created, modified, and then immediately used to `build` a final ChunkLightData object.

- **Destruction:** The builder becomes eligible for garbage collection as soon as the `build` method returns. The internal ByteBuf it manages is discarded; a new, compacted buffer is allocated for the resulting ChunkLightData object.

## Internal State & Concurrency
- **State:** The ChunkLightDataBuilder is highly **mutable**. Its primary state consists of the `light` ByteBuf, which contains the octree data, and the `currentSegments` BitSet, which tracks allocated 17-byte node slots within the buffer. Every call to a `setLight` method can potentially modify both the buffer content and the segment allocation map by splitting or collapsing octree nodes.

- **Thread Safety:** This class is **not thread-safe**. All operations directly manipulate the internal ByteBuf and BitSet without any synchronization. Concurrent access from multiple threads will corrupt the octree structure, leading to data loss, incorrect lighting, and potentially unrecoverable runtime exceptions. All interactions with a ChunkLightDataBuilder instance **must** be confined to a single thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ChunkLightDataBuilder(changeId) | constructor | O(1) | Creates a new, empty builder for a chunk section's light map. |
| ChunkLightDataBuilder(lightData, changeId) | constructor | O(N) | Creates a builder by decompressing an existing ChunkLightData object. N is the number of nodes in the source octree. |
| setBlockLight(index, r, g, b) | void | O(log V) | Sets the RGB block light for a single block. V is the volume (32768). Complexity is logarithmic due to octree traversal. |
| setSkyLight(index, light) | void | O(log V) | Sets the sky light value for a single block. |
| build() | ChunkLightData | O(N) | Finalizes the builder, compacting the internal octree into a new, immutable ChunkLightData object. N is the number of nodes in the builder's octree. |

## Integration Patterns

### Standard Usage
The builder is used as part of a "create-modify-finalize" workflow, typically within a lighting engine or chunk generation task.

```java
// How a developer should normally use this
// 1. Create a builder, either new or from existing data.
ChunkLightDataBuilder builder = new ChunkLightDataBuilder(existingData, newChangeId);

// 2. Perform a series of lighting modifications.
for (int i = 0; i < 100; i++) {
    int index = calculateBlockIndex(i);
    builder.setSkyLight(index, (byte) 15);
    builder.setBlockLight(index, (byte) 10, (byte) 8, (byte) 0);
}

// 3. Finalize the changes into an immutable object.
ChunkLightData finalLightData = builder.build();

// The builder is now discarded and the finalLightData is used.
chunk.setLightData(finalLightData);
```

### Anti-Patterns (Do NOT do this)
- **Builder Reuse:** Do not hold a reference to the builder after calling `build`. The object is not designed for reuse and subsequent modifications are undefined behavior. Always create a new builder for each distinct lighting operation.
- **Concurrent Modification:** Never share a ChunkLightDataBuilder instance across multiple threads. The lack of internal locking will lead to data corruption.
- **Building from a Builder:** The constructor explicitly throws an `IllegalArgumentException` if you attempt to create a builder from another builder. This is a design enforcement, as the internal state of a builder is not in the compact format expected by the constructor.

## Data Pipeline
The ChunkLightDataBuilder acts as a stateful transformer within the world update pipeline. It converts discrete block update commands into a compressed data structure.

> Flow:
> Block Update Event -> Lighting Engine Calculation -> **ChunkLightDataBuilder** (receives multiple `setLight` calls) -> `build()` -> Final `ChunkLightData` -> Chunk Serialization -> Network Packet to Client

