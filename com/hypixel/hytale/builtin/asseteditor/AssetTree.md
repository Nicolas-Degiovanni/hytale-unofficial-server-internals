---
description: Architectural reference for AssetTree
---

# AssetTree

**Package:** com.hypixel.hytale.builtin.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetTree {
```

## Architecture & Concepts
The AssetTree is a server-side, in-memory representation of the file and directory structure of a Hytale asset pack. It serves as the authoritative model for the Asset Editor, abstracting the physical file system into a structured, queryable, and mutable data model.

Its primary role is to maintain a consistent view of an asset pack's contents, which is then synchronized with a remote Asset Editor client. The tree is logically partitioned into two distinct collections: **Server** assets and **Common** assets. This separation reflects the engine's own asset loading structure.

This class is designed for a concurrent environment. It employs a StampedLock to manage access to its internal state, optimizing for scenarios with frequent read operations (e.g., client state synchronization, file lookups) and less frequent, but atomic, write operations (e.g., creating, deleting, or renaming assets). All modifications to the tree are transactional from the perspective of an outside observer.

The AssetTree is not merely a static snapshot; it is a dynamic model that can be modified via methods like `ensureAsset` and `removeAsset`. These changes are typically driven by user actions within the Asset Editor, which are communicated to the server and applied to the tree using the `applyAssetChanges` method.

## Lifecycle & Ownership
-   **Creation:** An AssetTree instance is created when a server-side process begins an editing session for a specific asset pack. The constructor is called with the root path of the pack. The more comprehensive constructor, which accepts a collection of AssetTypeHandlers, immediately triggers the `load` method, which recursively scans the file system to populate the initial state.

-   **Scope:** The object's lifetime is tightly coupled to an active Asset Editor session for a single asset pack, identified by its `packKey`. It persists as long as the pack is being managed by the server.

-   **Destruction:** The AssetTree is a standard Java object with no explicit destruction or `close` method. It becomes eligible for garbage collection when the corresponding editor session ends and all references to the instance are released.

## Internal State & Concurrency
-   **State:** The class is highly stateful and mutable. Its core state is composed of two `ObjectArrayList` collections: `serverAssets` and `commonAssets`. These lists contain `AssetEditorFileEntry` objects and are kept sorted by path to enable efficient binary search lookups. Immutable state, such as `rootPath` and `packKey`, is established at construction time.

-   **Thread Safety:** This class is thread-safe. All public methods that read or modify the internal asset lists (`serverAssets`, `commonAssets`) are protected by a `java.util.concurrent.locks.StampedLock`. This choice of lock indicates an architecture optimized for high-read, low-write contention. Read operations use an optimistic or shared read lock, while write operations acquire an exclusive write lock, ensuring atomicity and preventing data corruption from concurrent modifications.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| replaceAssetTree(AssetTree assetTree) | void | O(1) | Atomically replaces the internal asset lists with those from another AssetTree instance. Requires a full write lock. |
| sendPackets(EditorClient editorClient) | void | O(N) | Serializes the entire asset tree into network packets and sends them to the specified client. This is a read-only operation. |
| isDirectoryEmpty(Path path) | boolean | O(log N) | Performs a binary search to check if a given directory path has any child entries in the tree. |
| ensureAsset(Path path, boolean isDirectory) | AssetEditorFileEntry | O(N) | Creates a new file or directory entry in the tree if it does not exist. This is a write operation and may require shifting list elements. |
| getAssetFile(Path path) | AssetEditorFileEntry | O(log N) | Retrieves an asset entry by its path using a binary search. Returns null if not found. |
| removeAsset(Path path) | AssetEditorFileEntry | O(N) | Removes a file or an entire directory subtree from the tree. This is a write operation that shifts list elements. |
| applyAssetChanges(Map created, Map modified) | void | O(M * N) | Applies a batch of modifications (creations, deletions, renames) to the tree. This is the primary entry point for updating state. |

## Integration Patterns

### Standard Usage
The AssetTree is managed by a higher-level service responsible for an Asset Editor session. The typical flow involves creating the tree, loading it from disk, and then synchronizing it with a client. Subsequent changes are applied in batches.

```java
// 1. Create and load the tree when an editor session starts
Collection<AssetTypeHandler> handlers = ...;
AssetTree assetTree = new AssetTree(packRootPath, "my_pack", false, false, handlers);

// 2. Send the initial state to the connected client
EditorClient client = ...;
assetTree.sendPackets(client);

// 3. Later, apply changes received from the client or another source
Map<Path, ModifiedAsset> createdDirs = ...;
Map<Path, ModifiedAsset> modifiedFiles = ...;
assetTree.applyAssetChanges(createdDirs, modifiedFiles);

// 4. Re-synchronize the client with the new state
assetTree.sendPackets(client);
```

### Anti-Patterns (Do NOT do this)
-   **Unsorted State:** The internal binary search algorithms depend on the `serverAssets` and `commonAssets` lists being lexicographically sorted by path. Manually modifying these lists or replacing them with an unsorted tree via `replaceAssetTree` will break lookups and lead to unpredictable behavior.
-   **Lock Contention:** Avoid performing long-running or blocking I/O operations while holding the write lock. The `load` method, for instance, is correctly invoked only from the constructor before the object is shared across threads.
-   **Direct List Access:** Never access the internal `serverAssets` or `commonAssets` lists from another thread without acquiring the instance's `StampedLock`. Doing so bypasses all concurrency controls and will lead to `ConcurrentModificationException` or silent data corruption.

## Data Pipeline
The AssetTree acts as a central hub for asset structure information, transforming data between the file system, network packets, and in-memory modification sets.

> **Initial Load Flow:**
> File System -> `Files.walkFileTree` -> Raw Path List -> **AssetTree** (Populates `serverAssets` and `commonAssets`) -> Internal Sort

> **Client Synchronization Flow:**
> **AssetTree** (Read-locked access to lists) -> `sendPackets` -> `AssetEditorAssetListSetup` Packet -> Network Handler -> Editor Client UI

> **Modification Flow:**
> Editor Client Action -> Network Packet -> Server Handler -> `ModifiedAsset` Map -> `applyAssetChanges` -> **AssetTree** (Write-locked update to lists)

