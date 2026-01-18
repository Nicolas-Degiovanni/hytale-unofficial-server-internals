---
description: Architectural reference for IndexedStorageFile
---

# IndexedStorageFile

**Package:** com.hypixel.hytale.storage
**Type:** Transient

## Definition
```java
// Signature
public class IndexedStorageFile implements Closeable {
```

## Architecture & Concepts

The IndexedStorageFile is a high-performance, low-level file format and manager designed for random access to a collection of variable-sized binary blobs. It consolidates potentially thousands of small data chunks into a single, highly-structured file, mitigating filesystem inefficiencies related to managing numerous small files.

Its architecture is analogous to a simplified, file-based key-value store where the key is an integer index and the value is a binary blob. The core design principles are:

-   **Indexed Access:** A fixed-size header is followed by a memory-mapped index table. This table contains direct pointers (segment indices) to the location of each blob within the file, enabling O(1) lookup complexity for any given blob's location.
-   **Segmented Allocation:** The remainder of the file is divided into fixed-size blocks called segments. A blob can span multiple contiguous segments. This strategy, a form of slab allocation, simplifies space management and reduces fragmentation. A central BitSet, `usedSegments`, tracks allocated segments.
-   **On-the-Fly Compression:** All blobs are compressed using Zstandard (Zstd) before being written to a segment. The original and compressed sizes are stored in a small header preceding each blob's data, allowing for efficient decompression into a correctly-sized buffer.
-   **Fine-Grained Concurrency:** The system is engineered for high-throughput, multi-threaded access. It employs a sophisticated locking strategy using StampedLocks. Each entry in the blob index has its own lock, allowing concurrent reads and writes to different blobs. Segment allocation is also managed with a combination of locks to prevent race conditions during writes.

This class serves as a foundational storage layer, suitable for managing game assets, world chunk data, or any scenario requiring persistent, fast, random access to compressed binary data.

### Lifecycle & Ownership

-   **Creation:** An IndexedStorageFile is never instantiated directly via its constructor. The object is created exclusively through the static `open` factory methods. These methods handle the logic for creating a new storage file, opening an existing one, or performing a data migration from a legacy format (v0).
-   **Scope:** The object's lifetime is directly tied to the underlying FileChannel it manages. It represents an active, open file handle and persists as long as a reference is held.
-   **Destruction:** The caller has **strict ownership** and is responsible for invoking the `close` method. As this class implements `Closeable`, the recommended approach is a `try-with-resources` block to guarantee that the file channel is closed and the memory-mapped buffer is unmapped.

**WARNING:** Failure to call `close` will result in leaked file handles and native memory, as the `MappedByteBuffer` is cleaned using `Unsafe` utilities.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. Its primary state is the content of the file on disk. In memory, it maintains:
    -   File metadata read from the header (e.g., `blobCount`, `segmentSize`).
    -   `mappedBlobIndexes`: A MappedByteBuffer providing a direct, mutable view into the file's index table. Writes to this buffer are writes to the file.
    -   `usedSegments`: A BitSet that acts as the allocation map for all data segments in the file.
    -   `indexLocks` and `segmentLocks`: Arrays of StampedLocks that guard access to the index table and data segments, respectively.

-   **Thread Safety:** This class is thread-safe, designed for concurrent access.
    -   **Index Locking:** Access to each blob is controlled by a dedicated lock in the `indexLocks` array. This fine-grained approach allows two threads to operate on two different blobs simultaneously without contention.
    -   **Segment Allocation Locking:** The `findFreeSegment` method performs a complex locking dance. It first acquires a read lock on `usedSegmentsLock` to scan for free space. Once a candidate range is found, it attempts to acquire write locks on each individual segment in that range. This prevents race conditions where two threads attempt to claim the same free segments.
    -   **Optimistic Reads:** The use of StampedLock's `tryOptimisticRead` pattern is prevalent, providing a low-overhead path for reads that avoids the full cost of a lock acquisition if no concurrent writes are occurring.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| open(path, ...) | IndexedStorageFile | O(N) | Factory method to create or open a storage file. O(N) on first open due to scanning the index to build the `usedSegments` map. |
| readBlob(blobIndex) | ByteBuffer | O(S) | Reads, decompresses, and returns a blob. Complexity is proportional to the compressed size (S) of the blob. |
| writeBlob(blobIndex, src) | void | O(S + F) | Compresses and writes a blob. Complexity is proportional to source size (S) for compression and the number of file segments (F) to find a free slot. |
| removeBlob(blobIndex) | void | O(1) | Marks a blob's segments as free and clears its index entry. A very fast metadata-only operation. |
| keys() | IntList | O(N) | Returns a list of all blob indices that are currently in use. Requires a full scan of the index table. |
| close() | void | O(1) | Closes the file channel and releases all native resources. Critical for preventing resource leaks. |

## Integration Patterns

### Standard Usage

The class implements `Closeable` and must be managed carefully to prevent resource leaks. The `try-with-resources` statement is the correct and safest way to ensure `close` is always called.

```java
// How a developer should normally use this
Path storagePath = Paths.get("./world.dat");

// Always use try-with-resources to ensure the file is closed
try (IndexedStorageFile storage = IndexedStorageFile.open(storagePath, StandardOpenOption.CREATE, StandardOpenOption.READ, StandardOpenOption.WRITE)) {
    // Prepare some data
    ByteBuffer myData = ByteBuffer.wrap("Hello, World!".getBytes(StandardCharsets.UTF_8));

    // Write data to blob index 5
    storage.writeBlob(5, myData);

    // Read the data back
    ByteBuffer retrievedData = storage.readBlob(5);
    // ... process retrievedData

    // Remove the blob
    storage.removeBlob(5);

} catch (IOException e) {
    // Handle I/O errors
    e.printStackTrace();
}
```

### Anti-Patterns (Do NOT do this)

-   **Forgetting to Close:** The most critical anti-pattern. Not using a `try-with-resources` block or manually calling `close` in a `finally` block will leak file handles and native memory.
-   **External File Modification:** This class assumes it has exclusive control over the file's structure. Modifying the file with external tools while it is open will lead to corruption and undefined behavior.
-   **Ignoring IOExceptions:** All methods that interact with the disk can throw `IOException`. These must be handled properly, as a failed write could leave the file in an inconsistent state.
-   **Excessive `keys()` Calls:** The `keys()` method iterates the entire blob index. Calling it frequently in a performance-sensitive loop should be avoided.

## Data Pipeline

The flow of data for writing and reading operations is distinct and highly optimized.

### Write Pipeline
> Flow:
> `ByteBuffer` (uncompressed) -> Zstd Compression -> `ByteBuffer` (compressed, with size headers) -> **findFreeSegment** (allocates space) -> `FileChannel.write` (writes blob to segments) -> `MappedByteBuffer.putInt` (updates index pointer)

### Read Pipeline
> Flow:
> Blob Index -> `MappedByteBuffer.getInt` (reads segment pointer) -> `FileChannel.read` (reads compressed blob from segments) -> `ByteBuffer` (compressed) -> Zstd Decompression -> `ByteBuffer` (original uncompressed data)

