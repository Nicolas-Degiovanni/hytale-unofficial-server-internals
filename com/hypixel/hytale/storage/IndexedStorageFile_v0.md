---
description: Architectural reference for IndexedStorageFile_v0
---

# IndexedStorageFile_v0

**Package:** com.hypixel.hytale.storage
**Type:** Transient Resource Handle

## Definition
```java
// Signature
@Deprecated
public class IndexedStorageFile_v0 implements Closeable {
```

## Architecture & Concepts

The IndexedStorageFile_v0 class provides a low-level, thread-safe, and indexed storage system for binary large objects (blobs) within a single file. It is a foundational component for systems requiring fast, concurrent access to chunked or asset data directly from disk, such as world storage or asset databases.

**WARNING:** This class is deprecated. It represents a legacy implementation and should not be used for new development. Its architecture is preserved here for maintenance and migration purposes.

The core design revolves around a custom file format that divides the file into three primary regions:
1.  **File Header:** A fixed-size header containing a magic string (HytaleIndexedStorage), version number, total blob capacity, and segment size. This allows the system to validate and interpret the file structure upon opening.
2.  **Blob Index Table:** A memory-mapped table that provides O(1) lookup for any given blob. Each entry in the index is an integer that points to the *first segment* where the corresponding blob's data is stored. A value of 0 indicates the blob slot is empty. This table is memory-mapped via MappedByteBuffer for extremely high performance, avoiding repeated disk seeks for index lookups.
3.  **Segment Region:** The remainder of the file is a contiguous block of fixed-size data segments. Each blob is stored across a linked list of one or more segments. The first few bytes of each segment are reserved for a header that points to the index of the *next* segment in the chain.

Data written to a blob is first compressed using Zstd to reduce disk footprint. The system then allocates a sufficient number of segments to store the compressed data, writes the data, and updates the Blob Index Table to point to the start of the new segment chain.

A crucial feature is its crash-recovery mechanism. A secondary "temporary" index is maintained alongside the primary Blob Index Table. During a write operation, the new segment location is first written to the temporary index. Only after the entire write operation succeeds is the primary index updated. The `processTempIndexes` method runs on file open, checking for non-zero entries in the temporary index, which signify an incomplete write. It then follows the segment chain and deallocates the orphaned segments, ensuring file integrity after a crash.

## Lifecycle & Ownership

-   **Creation:** Instances are not created via a constructor. The class is exclusively instantiated through the static `open` factory methods. These methods handle the logic of either creating a new storage file or opening and validating an existing one.
-   **Scope:** An instance of IndexedStorageFile_v0 represents an active handle to a file on disk. It lives as long as the calling system requires access to that file. It holds critical system resources, including a FileChannel and a MappedByteBuffer.
-   **Destruction:** The `close` method **must** be called to release the file handle and properly unmap the memory buffer. Failure to do so will result in resource leaks and may prevent the file from being modified or deleted by other processes. The class implements `Closeable`, making it suitable for use in a try-with-resources block to guarantee cleanup.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. Its internal state, including the `fileChannel`, `mappedBlobIndexes`, and various lock arrays, is a direct representation of and handle to the underlying file on disk. Operations like `writeBlob` and `removeBlob` cause immediate and significant side effects on the filesystem.
-   **Thread Safety:** The class is designed to be thread-safe for concurrent read and write operations on *different blobs*. This is achieved through a fine-grained locking strategy:
    -   **indexLocks:** An array of `StampedLock`, with one lock per blob index. This allows two threads to write to different blobs (e.g., blob 5 and blob 10) simultaneously without contention.
    -   **segmentLocks:** A dynamically sized array of `StampedLock` used to manage the allocation of free segments during write operations.
    -   **nextSegmentIndexesLock:** A single `StampedLock` protecting the in-memory cache of the segment chain pointers.
    -   The implementation heavily favors `tryOptimisticRead` to minimize lock contention in read-heavy scenarios, falling back to full read locks only when the optimistic check fails.

**WARNING:** While operations on different blob indexes are safe, concurrently reading and writing to the *same* blob index from different threads will result in blocking and is not recommended for performance-critical paths.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| open(path, ...) | IndexedStorageFile_v0 | O(N) | Factory method to open or create a storage file. Complexity is proportional to file size during recovery scan. |
| readBlob(blobIndex) | ByteBuffer | O(S) | Reads, decompresses, and returns a blob. Complexity is proportional to the number of segments (S) the blob occupies. |
| writeBlob(blobIndex, src) | void | O(S) | Compresses and writes a blob. Finds free segments, writes data, and updates the index. Blocks until I/O is complete. |
| removeBlob(blobIndex) | void | O(S) | Deallocates all segments associated with a blob and clears its index entry. |
| keys() | IntList | O(B) | Returns a list of all blob indexes that are currently in use. Complexity is proportional to total blob capacity (B). |
| close() | void | O(1) | Releases the file handle and unmaps the memory buffer. Critical for resource cleanup. |

## Integration Patterns

### Standard Usage

The intended use is within a try-with-resources statement to ensure the file handle is always closed, even if exceptions occur.

```java
// Always use try-with-resources to guarantee the file is closed.
Path storagePath = Paths.get("./world.dat");

try (IndexedStorageFile_v0 storage = IndexedStorageFile_v0.open(storagePath, StandardOpenOption.CREATE, StandardOpenOption.READ, StandardOpenOption.WRITE)) {
    // Prepare data to be written
    ByteBuffer myData = ByteBuffer.wrap("Hello, World!".getBytes(StandardCharsets.UTF_8));

    // Write data to blob index 42
    storage.writeBlob(42, myData);

    // Read the data back
    ByteBuffer retrievedData = storage.readBlob(42);
    // ... process retrievedData
} catch (IOException e) {
    // Handle file I/O errors
    LOGGER.at(Level.SEVERE).withCause(e).log("Failed to operate on indexed storage file");
}
```

### Anti-Patterns (Do NOT do this)

-   **Ignoring Deprecation:** The primary anti-pattern is using this class in new systems. It is marked as `@Deprecated` and should be avoided in favor of newer storage solutions.
-   **Manual Resource Management:** Failing to use a try-with-resources block or forgetting to call `close()` in a `finally` block. This will lead to leaked file handles and memory-mapped buffers.
-   **Ignoring Threading Model:** Assuming that writes to the same blob index from multiple threads will be performant. The internal lock for that index will serialize access, creating a bottleneck.
-   **Excessive `keys()` Calls:** Calling the `keys()` method in a tight loop. The method iterates the entire blob index to find used slots and can be expensive for storage files with a large capacity.

## Data Pipeline

The flow of data through this component is distinct for read and write operations.

**Write Pipeline:**
> Flow:
> ByteBuffer (uncompressed) -> Zstd Compressor -> **IndexedStorageFile_v0** -> FileChannel -> Disk
> 1.  The source ByteBuffer is passed to `writeBlob`.
> 2.  The data is compressed in-memory using Zstd.
> 3.  The component finds and locks a contiguous range of free segments on disk.
> 4.  The compressed data is written across the allocated segments. Segment headers are updated to form a linked list.
> 5.  The memory-mapped blob index is updated to point to the first segment, making the data accessible.

**Read Pipeline:**
> Flow:
> Disk -> FileChannel -> **IndexedStorageFile_v0** -> Zstd Decompressor -> ByteBuffer (uncompressed)
> 1.  A call to `readBlob` provides a blob index.
> 2.  The component performs a near-instant lookup in the `mappedBlobIndexes` to find the first data segment.
> 3.  It follows the segment chain on disk, reading the compressed blob data into a direct ByteBuffer.
> 4.  The compressed data is passed to the Zstd decompressor.
> 5.  A new ByteBuffer containing the original, uncompressed data is returned to the caller.

