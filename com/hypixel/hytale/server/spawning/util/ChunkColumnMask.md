---
description: Architectural reference for ChunkColumnMask
---

# ChunkColumnMask

**Package:** com.hypixel.hytale.server.spawning.util
**Type:** Utility / Data Structure

## Definition
```java
// Signature
public class ChunkColumnMask {
```

## Architecture & Concepts
The ChunkColumnMask is a specialized, high-performance data structure designed to represent a fixed-size 32x32 grid of chunk columns. Its primary purpose is to efficiently track a set of columns, enabling quick lookups to determine if a specific column is "active" or "marked" for a given operation.

Internally, it is implemented as a wrapper around a Java **BitSet** of size 1024 (32 * 32). This design choice is critical for server performance, as it offers significant memory savings compared to a traditional boolean array. A **BitSet** stores each flag using a single bit, consuming approximately 128 bytes, whereas a boolean array would require at least 1024 bytes.

The class abstracts the one-dimensional nature of the **BitSet** by providing a two-dimensional API using (x, z) coordinates. It relies on the **ChunkUtil.indexColumn** method to translate these 2D coordinates into a 1D index suitable for the underlying **BitSet**.

A key architectural feature is the use of a bitwise AND mask (`& 1023`) on all index-based operations. This ensures that any provided index is wrapped to fit within the 0-1023 range, preventing **IndexOutOfBoundsException** errors. While this adds robustness, it can also mask logical errors if coordinates outside the intended 32x32 region are used.

Given its package location within `server.spawning.util`, its primary role is to support server-side systems like mob spawning, which need to track and query large regions of chunk columns with minimal performance overhead.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via `new ChunkColumnMask()`. This class is a plain Java object and is not managed by a service locator or dependency injection framework. It is typically created on the stack or as a member field of a higher-level system that requires its functionality.
- **Scope:** The lifetime is ephemeral and tied to the owning object or operation. For example, a mask might be created to track valid spawning locations for a single server tick and then be discarded. It is not intended to persist across sessions.
- **Destruction:** The object is managed by the Java garbage collector. No manual cleanup or destruction methods are required.

## Internal State & Concurrency
- **State:** The ChunkColumnMask is a mutable, stateful object. Its core state is the set of bits stored within the internal **BitSet**. The `final` keyword on the `columns` field only prevents the **BitSet** object itself from being replaced; the contents of the **BitSet** can be freely modified.
- **Thread Safety:** This class is **not thread-safe**. The underlying `java.util.BitSet` is not synchronized. Concurrent read and write operations from multiple threads will lead to race conditions, data corruption, and non-deterministic behavior.

**WARNING:** All interactions with a ChunkColumnMask instance must be confined to a single thread or be protected by external synchronization mechanisms. It is designed for use within single-threaded contexts, such as the main server game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| copyFrom(src) | void | O(N) | Copies the entire state from a source mask. |
| isEmpty() | boolean | O(1) | Checks if any bits are set. |
| clear() | void | O(N) | Clears all bits in the mask, setting them to false. |
| set() | void | O(N) | Sets all 1024 bits in the mask to true. |
| get(x, z) | boolean | O(1) | Checks if the bit for the specified column is set. |
| set(x, z) | void | O(1) | Sets the bit for the specified column to true. |
| clear(x, z) | void | O(1) | Clears the bit for the specified column. |
| nextSetBit(fromIndex) | int | O(N) | Finds the next set bit, for efficient iteration. |
| cardinality() | int | O(N) | Returns the total number of set bits. |

*Complexity N refers to the number of 64-bit longs used by the internal BitSet (1024 / 64 = 16).*

## Integration Patterns

### Standard Usage
The class is intended for direct instantiation and manipulation by systems that operate on grids of chunks, such as a spawner.

```java
// A system needs to determine which chunks in a 32x32 area are valid for an operation.
ChunkColumnMask validColumns = new ChunkColumnMask();

// Mark specific columns based on some logic
validColumns.set(5, 10);
validColumns.set(5, 11);

// Later, check if a column is valid before proceeding
int targetX = 5;
int targetZ = 10;
if (validColumns.get(targetX, targetZ)) {
    // Proceed with operation for this column...
}
```

### Anti-Patterns (Do NOT do this)
- **Multi-threaded Access:** Do not share a single ChunkColumnMask instance across multiple threads without external locking. This will corrupt the internal state.
- **Ignoring Coordinate Wrapping:** Do not pass coordinates outside the logical 0-31 range and expect an error. The internal bitmask (`& 1023`) will silently wrap the coordinates, potentially leading to hard-to-find bugs where `set(32, 0)` incorrectly modifies the state of column `(0, 0)`.
- **Unnecessary Re-creation:** For performance-critical loops, avoid creating a `new ChunkColumnMask()` on every iteration. Instead, reuse a single instance and call `clear()` to reset its state.

## Data Pipeline
The ChunkColumnMask is not a data pipeline component itself; rather, it acts as a stateful filter or set used by other systems within a pipeline. It holds state that other components query to make decisions.

> **Example Flow: Mob Spawning**
>
> Spawning System -> Query Player Locations -> **ChunkColumnMask (marks loaded/active chunks)** -> Filter Potential Spawn Points -> Spawn Mob

