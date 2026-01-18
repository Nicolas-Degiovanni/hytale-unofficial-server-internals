---
description: Architectural reference for StandardDataSource
---

# StandardDataSource

**Package:** com.hypixel.hytale.builtin.asseteditor.datasource
**Type:** Transient

## Definition
```java
// Signature
public class StandardDataSource implements DataSource {
```

## Architecture & Concepts
The StandardDataSource is the canonical, filesystem-backed implementation of the DataSource interface. It serves as the primary bridge between the logical operations of the Hytale Asset Editor and the physical asset files stored on disk. Each instance of StandardDataSource corresponds to a single, discrete asset pack, such as a core game resource pack or a user-generated mod.

Its core architectural purpose is to provide a robust and transactional abstraction over raw file I/O. It handles critical tasks such as resolving relative asset paths to absolute filesystem paths, managing CRUD operations (Create, Read, Update, Delete), and logging I/O failures.

A key subsystem within StandardDataSource is its change tracking and persistence mechanism. It maintains an in-memory, bounded cache of recently modified assets. This cache is periodically flushed to a dedicated index file on disk. This ensures that metadata about recent changes, such as which user last edited a file, persists across server restarts.

Furthermore, it implements a sophisticated de-bouncing system via the `shouldReloadAssetFromDisk` method. When the Asset Editor saves a file, this class temporarily tracks the file's new hash. This prevents the server's file-watching service from immediately and redundantly reloading the same asset that the editor just wrote, which would otherwise cause significant performance issues and potential race conditions.

## Lifecycle & Ownership
- **Creation:** A StandardDataSource is instantiated by a higher-level management service, likely the central AssetEditor service, for each asset pack that is loaded and made available for editing. It is not intended for direct instantiation by general-purpose code.
- **Scope:** The object's lifetime is tightly coupled to the asset pack it represents. It is created when an asset pack is loaded into the editor's context and persists until the pack is unloaded or the server shuts down.
- **Destruction:** The `shutdown` method must be invoked by the owning service. This triggers a final, blocking save of the recent modifications index to disk, ensuring no data from the current session is lost. Failure to call `shutdown` is a critical error that will lead to data loss.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains an in-memory map of `modifiedAssets`, a map of `editorSaves` to track recent writes, and the `assetTree` which represents the directory structure. Its state is a direct reflection of a folder on the filesystem, augmented with session-specific metadata.
- **Thread Safety:** StandardDataSource is designed to be thread-safe. It leverages concurrent collections like ConcurrentHashMap and atomic primitives like AtomicBoolean for its primary state. Critical sections, such as updating the deque of recent file saves in `shouldReloadAssetFromDisk`, are protected by explicit synchronization blocks. This allows safe interaction from multiple threads, including network threads processing editor commands and the dedicated background thread that saves the modification index.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start() | void | O(N) | Initializes the component by loading the modification index from disk and starting the periodic save scheduler. Must be called before any other operation. |
| shutdown() | void | O(M) | Cancels the save scheduler and performs a final, blocking flush of the modification index to disk. |
| updateAsset(path, bytes, client) | boolean | O(L) | Writes the provided bytes to the specified asset path, overwriting existing content. Tracks the modification. L is the size of the byte array. |
| createAsset(path, bytes, client) | boolean | O(L) | Creates a new asset at the specified path with the given content. Tracks the modification. |
| deleteAsset(path, client) | boolean | O(1) | Deletes the asset at the specified path from the filesystem. Tracks the modification. |
| getAssetBytes(path) | byte[] | O(L) | Reads and returns the complete byte content of an asset. Returns null on I/O failure. |
| shouldReloadAssetFromDisk(path) | boolean | O(1) | Checks if an asset needs to be reloaded from disk. Returns false if the current content on disk matches a recent save from the editor. |
| getAssetTree() | AssetTree | O(1) | Returns a reference to the in-memory representation of the asset pack's file hierarchy. |

## Integration Patterns

### Standard Usage
The StandardDataSource is not meant to be retrieved directly. It is managed by a central service that exposes its functionality. A typical interaction involves requesting a file operation for a specific asset pack.

```java
// A higher-level service would manage and provide the correct DataSource instance
// for a given asset pack key.
DataSource dataSource = assetEditor.getDataSourceForPack("my_mod");

// Update an asset's content
boolean success = dataSource.updateAsset(
    Path.of("textures/blocks/stone.png"),
    newTextureData,
    editorClient
);

if (!success) {
    // The underlying I/O operation failed; handle the error.
    LOGGER.at(Level.SEVERE).log("Failed to save asset!");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new StandardDataSource()`. The lifecycle of this object is critical and must be managed by the core Asset Editor service to ensure `start()` and `shutdown()` are called correctly. Unmanaged instances will not persist their state and will leak resources.
- **Forgetting Shutdown:** Failure to call `shutdown()` on server or asset pack unload will result in the loss of all modification metadata generated during the session.
- **Path Traversal:** Do not construct asset paths with relative segments like ".." to access files outside the designated `rootPath`. The implementation relies on path resolution within its root directory.

## Data Pipeline
The data flow for a typical asset modification involves two parallel pipelines: one for the immediate filesystem write and another for persisting the change metadata.

> **Asset Write Pipeline:**
> Editor Client Command -> Network Service -> Asset Editor -> **StandardDataSource.updateAsset()** -> java.nio.Files.write() -> Filesystem

> **Metadata Persistence Pipeline:**
> **StandardDataSource.updateAsset()** -> Adds `ModifiedAsset` to `modifiedAssets` map -> Sets `indexNeedsSaving` flag to true -> HytaleServer.SCHEDULED_EXECUTOR (triggers every 1 minute) -> **StandardDataSource.saveRecentModifications()** -> Writes BSON to `recentAssetEdits_{packKey}.json`

