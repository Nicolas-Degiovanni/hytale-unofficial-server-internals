---
description: Architectural reference for ChunkUtil
---

# ChunkUtil

**Package:** com.hypixel.hytale.math.util
**Type:** Utility

## Definition
```java
// Signature
public class ChunkUtil {
```

## Architecture & Concepts

The ChunkUtil class is a foundational, low-level utility that defines the geometric and spatial rules of the Hytale world. It serves as the single source of truth for constants related to world structure, such as chunk dimensions, height limits, and bit masks. Its primary function is to provide a suite of highly-optimized, static methods for coordinate transformation and data indexing.

This class is a cornerstone of the world engine, sitting at the boundary between abstract world coordinates and the concrete memory layout of chunk data. By centralizing these calculations, the engine ensures consistency and performance across all systems that interact with world geometry, including rendering, physics, networking, and world generation.

The design heavily favors bitwise operations (shifting, masking) over standard arithmetic for maximum performance. These methods are expected to be called millions of time per second in tight loops, and their efficiency is critical to the engine's overall performance. It effectively translates three-dimensional spatial concepts into one-dimensional array indices used for data storage.

## Lifecycle & Ownership

-   **Creation:** The ChunkUtil class is never instantiated. Its private constructor prevents the creation of objects. The class is loaded into the JVM by the ClassLoader upon its first use.
-   **Scope:** Application-level. As a collection of static constants and methods, it exists for the entire lifetime of the application.
-   **Destruction:** The class is unloaded from the JVM when the application terminates. No manual resource management is required.

## Internal State & Concurrency

-   **State:** ChunkUtil is completely stateless and immutable. It consists solely of `static final` constants and pure functions. The output of its methods depends exclusively on the input arguments.
-   **Thread Safety:** This class is inherently thread-safe. Its stateless, pure-function design guarantees that it can be safely and concurrently called from any thread without locks or other synchronization primitives. This is a critical architectural feature for a highly parallel game engine.

## API Surface

The public API consists of two main categories: geometric constants and coordinate transformation functions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SIZE | int | O(1) | The dimension of a chunk along the X and Z axes (32). |
| HEIGHT | int | O(1) | The total height of a chunk column in blocks (320). |
| indexChunk(x, z) | long | O(1) | Packs 2D integer chunk coordinates into a single long for use as a unique key. |
| chunkCoordinate(block) | int | O(1) | Converts a global block coordinate into a chunk coordinate via bit shifting. |
| indexBlock(x, y, z) | int | O(1) | Converts 3D local-to-chunk coordinates into a 1D array index for a 32x32x32 section. |
| xOfChunkIndex(index) | int | O(1) | Extracts the X coordinate from a long chunk index. |
| shortToByteArray(data) | byte[] | O(N) | Serializes a short array into a little-endian byte array for network or disk I/O. |

## Integration Patterns

### Standard Usage

ChunkUtil is used whenever a system needs to translate between different spatial reference frames. The most common use case is finding the correct chunk for a given world position and then calculating the local index within that chunk's data array.

```java
// A system needs to get the block type at a global world position.
int globalX = 485;
int globalY = 67;
int globalZ = -120;

// 1. Determine the host chunk's unique ID.
long chunkId = ChunkUtil.indexChunkFromBlock(globalX, globalZ);
Chunk targetChunk = world.getChunk(chunkId);

// 2. Calculate the block's local coordinates within the chunk.
int localX = globalX & ChunkUtil.SIZE_MASK;
int localY = globalY & ChunkUtil.SIZE_MASK;
int localZ = globalZ & ChunkUtil.SIZE_MASK;

// 3. Calculate the 1D index for the block data array.
int blockIndex = ChunkUtil.indexBlock(localX, localY, localZ);
BlockType type = targetChunk.getBlockType(blockIndex);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not attempt to create an instance of ChunkUtil. The class is not designed to be instantiated, and the private constructor will prevent it. All methods must be accessed statically.
-   **Coordinate System Mismatch:** **CRITICAL:** Never pass global block coordinates into a method expecting local-to-chunk coordinates (e.g., `indexBlock`). This will produce an incorrect array index, leading to out-of-bounds access, world corruption, or severe visual artifacts. Always mask global coordinates before calculating local indices.
-   **Floating-Point Imprecision:** Avoid passing raw floating-point values from physics or entity positions directly to integer-based methods. This can lead to off-by-one errors. Always use the provided `MathUtil.floor` or similar rounding methods before converting to an integer block coordinate.

## Data Pipeline

ChunkUtil is not a data processor in a traditional pipeline sense, but rather a transformation utility used at various stages within other pipelines.

**Example 1: World Rendering Pipeline**

> Flow:
> Camera Position (Vector3d) -> `MathUtil.floor` -> Global Block Coordinate (int x, y, z) -> **ChunkUtil.indexChunkFromBlock** -> Chunk ID (long) -> Retrieve Chunk Data -> **ChunkUtil.indexBlock** -> Block Data Index -> Render Mesh

**Example 2: Network Serialization Pipeline**

> Flow:
> In-Memory Chunk Data (short[] blockIDs) -> **ChunkUtil.shortToByteArray** -> Serialized Payload (byte[]) -> Network Packet -> Client

---

