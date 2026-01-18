---
description: Architectural reference for BitFieldArr
---

# BitFieldArr

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.palette
**Type:** Utility / Data Structure

## Definition
```java
// Signature
public class BitFieldArr {
```

## Architecture & Concepts
BitFieldArr is a low-level, high-performance data structure designed for extreme memory optimization within the server's world representation. It implements a packed array of integers, where each integer occupies a custom, fixed number of bits.

This class is a foundational component of the **Chunk Palette** system. In a voxel world, a single chunk can contain thousands of blocks, but often only a small number of unique block *types*. Instead of storing a full 32-bit or 16-bit block ID for every position, the palette system maps each unique block type to a small local index. BitFieldArr is the storage mechanism for these small indices.

For example, if a chunk section contains only 15 unique block types, each block's state can be represented by a 4-bit index (2^4 = 16) instead of a 16-bit global ID. This results in a 75% reduction in memory usage for block data storage, which is critical for server performance and scalability. The class achieves this by tightly packing these bit-level values into a contiguous byte array, spanning values across byte boundaries where necessary.

## Lifecycle & Ownership
BitFieldArr is a transient object whose lifecycle is strictly controlled by a parent container.

-   **Creation:** Instantiated directly by higher-level world data structures, typically a Palette or ChunkSection, during world generation or chunk loading. The constructor requires the number of bits per entry and the total number of entries, which defines the storage capacity.
-   **Scope:** The object's lifetime is tightly coupled to its owner. It exists only as long as the parent chunk or palette data is held in memory.
-   **Destruction:** There is no explicit destruction or cleanup method. A BitFieldArr instance is eligible for garbage collection as soon as its parent container is dereferenced and collected, for example, when a chunk is unloaded from memory.

## Internal State & Concurrency
-   **State:** The internal state is **mutable**. The core state is the private byte array that holds the packed data. The configuration fields, bits and length, are immutable after construction. The get and set methods directly manipulate the internal byte array.
-   **Thread Safety:** This class is **not thread-safe**. All read and write operations on the underlying byte array are unsynchronized. Concurrent access from multiple threads will result in race conditions, data corruption, and undefined behavior. Any system using BitFieldArr must implement its own external locking or ensure access is confined to a single thread, such as the main world-tick thread.

## API Surface
The public API provides direct, low-level access to the packed data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BitFieldArr(bits, length) | constructor | O(N) | Allocates the internal byte array. Complexity is proportional to the size of the array. |
| get(index) | int | O(1) | Reads the integer value at the specified index. Involves complex bit-shifting. |
| set(index, value) | void | O(1) | Writes the integer value at the specified index. Throws no exception if value exceeds bit capacity; data is truncated via bitwise operations. |
| getLength() | int | O(1) | Returns the number of entries the array can hold. |
| get() | byte[] | O(N) | Returns a **deep copy** of the internal byte array. Safe to mutate the returned array. |
| set(bytes) | void | O(N) | Overwrites the internal array with data from an external source. Used for deserialization. |
| copyFrom(other) | void | O(N) | Copies data from another BitFieldArr. **Warning:** Throws an exception if the bit or length configurations do not match. |

*Note on Complexity: Operations are technically proportional to the number of bits per entry, but this is a small, fixed constant, making them effectively O(1) for practical analysis.*

## Integration Patterns

### Standard Usage
BitFieldArr is not intended for general application use. It should only be used by systems managing chunk data that require dense, packed data storage. The typical user is a Palette implementation.

```java
// A Palette creates a BitFieldArr to store 4096 block indices,
// using 5 bits for each index.
int bitsPerEntry = 5;
int totalEntries = 4096;
BitFieldArr storage = new BitFieldArr(bitsPerEntry, totalEntries);

// Set the block at local coordinate (0,0,0) to palette index 3
storage.set(0, 3);

// Retrieve the index later
int paletteIndex = storage.get(0);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never access a single BitFieldArr instance from multiple threads without external synchronization. This is the most common and severe misuse of this class.
-   **Incorrect Bit Sizing:** Do not initialize the array with a `bits` value smaller than required for the range of values you intend to store. For example, using 4 bits to store the value 17 will result in silent data truncation, as `17 (10001)` becomes `1 (0001)`.
-   **Mismatched `copyFrom`:** The `copyFrom` method contains defective validation logic that throws an exception if the source and destination have the *same* dimensions. Do not use this method until its implementation is corrected. The intended use is to copy between arrays of identical size and bit-depth.

## Data Pipeline
BitFieldArr serves as the in-memory representation for paletted block data, acting as a bridge between game logic and serialized storage or network transport.

> **Deserialization Flow:**
> Network Packet / Disk File -> Raw byte[] -> **BitFieldArr.set(byte[])** -> In-Memory Chunk Representation

> **Serialization Flow:**
> In-Memory Chunk Representation -> **BitFieldArr.get()** -> Raw byte[] -> Network Packet / Disk File

> **Game Logic Flow:**
> World Tick -> Block Update Logic -> Palette Lookup -> **BitFieldArr.set(index, value)** -> State Change

