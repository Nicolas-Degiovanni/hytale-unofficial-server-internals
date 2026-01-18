---
description: Architectural reference for StorageManager
---

# StorageManager

**Package:** com.hypixel.hytale.storage
**Type:** Singleton

## Definition
```java
// Signature
public final class StorageManager implements IService, IShutdownable {
```

## Architecture & Concepts
The StorageManager is the centralized service responsible for all persistent data storage operations within the engine. It serves as a crucial abstraction layer between game systems and the underlying physical storage medium, which may include local files, SQLite databases, or platform-specific cloud storage.

Its primary architectural goals are:
1.  **Decoupling:** Game logic (e.g., saving player inventory, world state) should not be aware of the specifics of file paths, serialization formats, or database schemas.
2.  **Performance:** Disk I/O is a significant performance bottleneck. The StorageManager offloads all read and write operations to a dedicated, low-priority background thread pool to prevent stalling the main game loop.
3.  **Atomicity & Durability:** It provides transactional semantics for save operations, ensuring that data is either fully written or not at all, preventing data corruption from partial writes during a crash.

The manager operates on a key-value paradigm, where a unique string key maps to a serializable data object. It internally manages serialization (e.g., JSON, BSON), compression, and optional encryption before committing data to disk.

## Lifecycle & Ownership
- **Creation:** The StorageManager is instantiated once by the core EngineContext during the earliest stages of the application bootstrap sequence. It is a foundational service that must be available before most other game systems are initialized.
- **Scope:** The instance is a global singleton that persists for the entire application session. All access must be routed through the central service registry.
- **Destruction:** The shutdown method is invoked by the EngineContext during the application shutdown sequence. This triggers a graceful shutdown, ensuring all pending write operations in its internal queue are flushed to disk before the process terminates.

**WARNING:** Forcibly terminating the application without a graceful shutdown may result in the loss of data that is still pending in the write queue.

## Internal State & Concurrency
- **State:** The StorageManager is highly stateful and mutable. It maintains an internal command queue for pending I/O operations, a connection pool for database access, and an optional in-memory write-through cache (L1 cache) to coalesce frequent writes to the same key.
- **Thread Safety:** This class is fully thread-safe. Public API methods are designed to be non-blocking and can be safely invoked from any thread, including the main render thread. Concurrency is managed internally by dispatching all operations to a single-threaded I/O executor, which serializes all disk access and prevents race conditions. Callers receive a CompletableFuture to track the operation's asynchronous result.

## API Surface
The public contract is entirely asynchronous to enforce non-blocking I/O patterns.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| saveAsync(key, object) | CompletableFuture<Void> | O(1) | Asynchronously serializes and saves an object. The future completes upon successful write to disk or fails with an IOException. |
| loadAsync(key, type) | CompletableFuture<T> | O(1) | Asynchronously loads and deserializes an object. The future completes with the object or fails if the key is not found or a deserialization error occurs. |
| deleteAsync(key) | CompletableFuture<Boolean> | O(1) | Asynchronously deletes the data associated with the key. The future completes with true if the data existed and was deleted. |
| shutdown() | void | O(N) | Blocks the calling thread until all N pending I/O operations are completed. Invoked by the engine. |

## Integration Patterns

### Standard Usage
All interaction with the StorageManager should be asynchronous. Retrieve the service from the context and use chained callbacks or async-await patterns to handle the results without blocking the main thread.

```java
// Correctly save player data without blocking the game loop
StorageManager storage = context.getService(StorageManager.class);
CompletableFuture<Void> saveOperation = storage.saveAsync("player.inventory", playerData);

saveOperation.thenRun(() -> {
    // This code runs on the main thread after the save is complete
    log.info("Player inventory saved successfully.");
}).exceptionally(error -> {
    log.error("Failed to save inventory!", error);
    return null; // Handle the error
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new StorageManager(). This will bypass the engine's singleton management and create a rogue instance that will conflict with the primary one, leading to file locking errors and data corruption.
- **Blocking on the Main Thread:** Calling get() on a returned CompletableFuture from the main game thread is a severe performance error. It will freeze the client until the disk I/O operation is complete, causing noticeable stutter or a full application hang.

    ```java
    // ANTI-PATTERN: This will freeze the game!
    StorageManager storage = context.getService(StorageManager.class);
    storage.saveAsync("player.settings", settings).get(); // DO NOT DO THIS
    ```
- **Mutating Objects After Save:** Do not modify an object after passing it to saveAsync. The serialization may occur on a background thread at a later time. If the object is modified on the main thread concurrently, a race condition will occur, leading to inconsistent or corrupted data being saved. Pass an immutable copy if the object must be modified immediately.

## Data Pipeline
The flow of data for a write operation is managed to ensure safety and performance.

> Flow:
> Game System -> Calls **StorageManager.saveAsync(key, object)** -> Object and Key wrapped in a WriteCommand -> WriteCommand enqueued in internal ConcurrentLinkedQueue -> I/O Executor Thread dequeues command -> Object is serialized to byte array -> Byte array is written to disk -> CompletableFuture is completed.

