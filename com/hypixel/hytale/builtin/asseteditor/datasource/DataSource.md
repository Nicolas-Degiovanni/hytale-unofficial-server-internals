---
description: Architectural reference for the DataSource interface, the core abstraction for asset storage and retrieval in the Asset Editor.
---

# DataSource

**Package:** com.hypixel.hytale.builtin.asseteditor.datasource
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface DataSource {
    // ... methods
}
```

## Architecture & Concepts
The DataSource interface is a foundational abstraction within the Hytale Asset Editor framework. It defines a strict contract for how the editor interacts with underlying asset storage systems, effectively decoupling the editor's logic from the physical location and nature of the asset data. This is a classic application of the Strategy design pattern, allowing the system to swap out different storage backends—such as a local file system, a remote database, or a version control repository—without altering the core editor components.

Its primary responsibilities are:
1.  **Asset Hierarchy Management:** Providing a structured view of all assets via the AssetTree.
2.  **CRUD Operations:** Handling the creation, retrieval, modification, and deletion of both assets and directories.
3.  **Metadata Access:** Supplying metadata such as modification timestamps and path information.
4.  **Mutability Contract:** Explicitly declaring whether the underlying data store is read-only or writable via the isImmutable method.

All high-level editor operations that involve asset persistence are funneled through an active implementation of this interface.

### Lifecycle & Ownership
As an interface, DataSource itself has no lifecycle. The following describes the lifecycle of a concrete implementation, for example, a FileSystemDataSource.

-   **Creation:** A specific DataSource implementation is instantiated by the Asset Editor's primary bootstrap service during application startup. The choice of implementation is determined by engine configuration and launch parameters (e.g., local project mode vs. connected server mode).
-   **Scope:** The DataSource instance is a session-scoped singleton. It persists for the entire duration of the Asset Editor's runtime. A single, authoritative instance is held by a central service registry.
-   **Destruction:** The `shutdown` method is invoked by the Asset Editor's shutdown hook. This is a critical step for releasing system resources, such as file handles, network connections, or database locks. Failure to call `shutdown` will result in resource leaks.

## Internal State & Concurrency
-   **State:** The state is entirely managed by the implementing class and reflects the underlying storage medium. For a file-system-based implementation, the state is highly mutable and directly corresponds to the files and directories on disk. Implementations are expected to cache the AssetTree structure after the initial `loadAssetTree` call for performance.
-   **Thread Safety:** Implementations of DataSource are **not guaranteed to be thread-safe**. File system and network operations are typically blocking and stateful. All interactions with a DataSource instance should be marshaled onto a single, dedicated thread, usually the main editor or UI thread.

    **Warning:** Concurrent calls to methods like `updateAsset` and `moveDirectory` from multiple threads will lead to race conditions, data corruption, and unpredictable behavior. The `EditorClient` parameter in mutation methods further implies a single-user, single-thread context for write operations.

## API Surface
The public API defines a complete set of operations for managing a hierarchical asset store.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start() | void | O(1) | Initializes the data source, preparing it for operations. |
| shutdown() | void | O(1) | Releases all acquired resources. Must be called on exit. |
| loadAssetTree(...) | AssetTree | O(N) | Scans the entire data source and builds a hierarchical tree of all assets. N is the total number of assets. |
| getAssetBytes(path) | byte[] | O(S) | Retrieves the raw byte content of an asset. S is the size of the asset. |
| updateAsset(path, bytes, client) | boolean | O(S) | Writes new byte content to an existing asset. |
| createAsset(path, bytes, client) | boolean | O(S) | Creates a new asset at the specified path with the given content. |
| isImmutable() | boolean | O(1) | Critical contract method. Returns true if the data source is read-only. |

## Integration Patterns

### Standard Usage
The Asset Editor framework retrieves the configured DataSource implementation from a central service registry. It is used to populate the UI and to execute user-driven file operations.

```java
// Correctly retrieve the session-scoped DataSource
DataSource assetSource = editorContext.getService(DataSource.class);

// Load the asset hierarchy to populate the UI tree
AssetTree tree = assetSource.loadAssetTree(assetTypeHandlers);
editorUI.populateTree(tree);

// Handle a user saving an asset
byte[] updatedData = getBytesFromEditor();
Path assetPath = getCurrentAssetPath();
boolean success = assetSource.updateAsset(assetPath, updatedData, editorClient);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never instantiate a DataSource implementation directly (e.g., `new FileSystemDataSource()`). The system's bootstrap process is responsible for creating and configuring the correct instance for the session. Always retrieve it from the context or service registry.
-   **Ignoring Immutability:** Calling mutation methods like `createAsset` or `deleteDirectory` without first checking `isImmutable()`. Attempting to write to a read-only source is a programmatic error and may throw an exception or fail silently.
-   **Omitting Shutdown:** Failing to ensure `shutdown()` is called during the editor's exit sequence. This is a primary cause of resource leaks.
-   **Path Assumption:** Assuming the Path objects used by the DataSource correspond directly to local file system paths. The Path is an abstract identifier; the `getFullPathToAssetData` method should be used to resolve it to a concrete location if necessary.

## Data Pipeline
The DataSource serves as the bridge between logical asset operations and the physical storage layer.

**Read Flow:**
> User Action (Select Asset) -> Editor UI Event -> AssetEditorService::getAssetBytes -> **DataSource::getAssetBytes(path)** -> Filesystem/Network Read -> Raw Bytes -> AssetTypeHandler::decode -> Rendered in Editor

**Write Flow:**
> User Action (Save Asset) -> Editor UI Event -> AssetTypeHandler::encode -> Raw Bytes -> AssetEditorService::updateAsset -> **DataSource::updateAsset(path, bytes, client)** -> Filesystem/Network Write -> Storage Medium

