---
description: Architectural reference for IndexedStorageChunkStorageProvider
---

# IndexedStorageChunkStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.world.storage.provider
**Type:** Factory / Strategy

## Definition
```java
// Signature
public class IndexedStorageChunkStorageProvider implements IChunkStorageProvider {
```

## Architecture & Concepts
The IndexedStorageChunkStorageProvider is a concrete implementation of the IChunkStorageProvider strategy interface. It serves as a factory for creating chunk loaders and savers that persist world data to disk using Hytale's proprietary IndexedStorageFile format. This format is highly optimized for spatial data, grouping chunks into larger region files to minimize file handle usage and improve I/O performance.

This provider is the bridge between the logical concept of a ChunkStore and the physical file-based persistence layer. Its core architectural components are:

- **Region-Based Storage:** Chunks are not stored in individual files. Instead, they are grouped into 32x32 chunk regions. Each region is stored in a single file on disk (e.g., `0.0.region.bin`), which is managed by an IndexedStorageFile instance. This strategy significantly reduces filesystem overhead.

- **Shared Cache:** The provider's primary responsibility is to create loaders and savers. These objects do not manage file handles directly. Instead, they interact with a central, shared **IndexedStorageCache** instance. This cache is a session-scoped resource that manages all open region file handles, preventing redundant file operations and providing a concurrent access layer.

- **Asynchronous I/O:** All load and save operations, exposed via the IChunkLoader and IChunkSaver interfaces, return a CompletableFuture. This ensures that all disk I/O is performed asynchronously, preventing the main server thread from blocking on slow disk operations.

- **ECS Integration:** The lifecycle of the underlying cache is managed by the **IndexedStorageCacheSetupSystem**, an Entity-Component-System (ECS) style system. This system hooks into the ChunkStore's initialization phase to correctly configure the storage path, demonstrating a tight integration with the game's core component model.

## Lifecycle & Ownership
- **Creation:** The IndexedStorageChunkStorageProvider itself is typically instantiated by the server's storage management layer during world initialization, based on server configuration. The loaders and savers it produces are then attached to the active ChunkStore.

- **Scope:** An instance of the provider is transient; it is used to create the loader/saver and can then be discarded. The critical state is held within the **IndexedStorageCache**, which is registered as a Resource on the ChunkStore. The cache persists for the entire lifetime of the World session.

- **Destruction:** The loaders and savers are closed when the ChunkStore is shut down. This triggers the close method on the shared IndexedStorageCache, which in turn iterates through all open IndexedStorageFile handles and closes them gracefully, ensuring all data is flushed to disk.

## Internal State & Concurrency
- **State:** The IndexedStorageChunkStorageProvider is **stateless**. All state is managed by the nested **IndexedStorageCache** class. The cache is highly **mutable**, maintaining a map of region coordinates to open IndexedStorageFile handles.

- **Thread Safety:** The provider itself is thread-safe due to its stateless nature. The stateful **IndexedStorageCache** is designed for concurrent use. It employs a Long2ObjectConcurrentHashMap to safely manage file handles, allowing multiple threads from the asynchronous I/O worker pool to open, read, write, and create region files without explicit locking from the caller's perspective. All I/O operations on the returned loaders and savers are thread-safe and can be invoked from any thread.

## API Surface
The public contract of the provider is minimal, acting purely as a factory.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLoader(Store) | IChunkLoader | O(1) | Creates a new IndexedStorageChunkLoader instance for reading chunk data. |
| getSaver(Store) | IChunkSaver | O(1) | Creates a new IndexedStorageChunkSaver instance for writing chunk data. |

## Integration Patterns

### Standard Usage
A developer or system will never interact with this provider directly. The server's world loading mechanism configures the ChunkStore to use this provider. The subsequent interactions are with the IChunkLoader and IChunkSaver interfaces obtained from the store.

The setup is handled internally by the ECS system.

```java
// This system automatically configures the cache path.
// It is added to the ChunkStore when the world is initialized.
public class IndexedStorageCacheSetupSystem extends StoreSystem<ChunkStore> {
    @Override
    public void onSystemAddedToStore(@Nonnull Store<ChunkStore> store) {
        World world = store.getExternalData().getWorld();
        // The cache is retrieved as a resource and configured.
        store.getResource(IndexedStorageCache.getResourceType()).path = 
            world.getSavePath().resolve("chunks");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create an IndexedStorageChunkLoader or IndexedStorageChunkSaver using `new`. They rely on being connected to a ChunkStore that has a properly initialized IndexedStorageCache resource. Failure to do so will result in NullPointerExceptions.

- **Cache Mismanagement:** Do not attempt to retrieve and manipulate the IndexedStorageCache resource directly. Its lifecycle is managed by the ChunkStore. Manually closing the cache will lead to I/O errors for all subsequent chunk operations.

## Data Pipeline
The provider establishes a clear and asynchronous pipeline for chunk data between the game state and the physical disk.

> **Save Flow:**
> Chunk ByteBuffer -> `IndexedStorageChunkSaver.saveBuffer()` -> CompletableFuture -> I/O Worker Thread -> **IndexedStorageCache**.getOrCreate() -> IndexedStorageFile.writeBlob() -> Filesystem

> **Load Flow:**
> Chunk Request -> `IndexedStorageChunkLoader.loadBuffer()` -> CompletableFuture -> I/O Worker Thread -> **IndexedStorageCache**.getOrTryOpen() -> IndexedStorageFile.readBlob() -> Chunk ByteBuffer

