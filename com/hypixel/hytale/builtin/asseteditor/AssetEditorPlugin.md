---
description: Architectural reference for AssetEditorPlugin
---

# AssetEditorPlugin

**Package:** com.hypixel.hytale.builtin.asseteditor
**Type:** Singleton

## Definition
```java
// Signature
public class AssetEditorPlugin extends JavaPlugin {
```

## Architecture & Concepts
The AssetEditorPlugin serves as the central, server-side orchestrator for all real-time asset editing functionality. As a core `JavaPlugin`, it is deeply integrated into the server's lifecycle and module system. Its primary responsibility is to act as the authoritative bridge between connected `EditorClient` instances and the server's underlying asset and filesystem abstractions, primarily the `AssetModule` and `DataSource` layers.

This plugin manages the entire lifecycle of an asset editing session, from the initial handshake and capability exchange with a client to the processing of complex, stateful operations like asset creation, modification, and deletion. It maintains a comprehensive view of the server's asset packs, their file trees, and their associated schemas.

Key architectural functions include:

*   **Session Management:** Tracks all connected `EditorClient`s, their permissions, and the specific assets they currently have open.
*   **Protocol Gateway:** Implements a suite of `handle...` methods that serve as the entry points for processing network packets sent from the editor client.
*   **State Synchronization:** Ensures that changes made by one client are validated, applied to the filesystem, loaded into the running game via the `AssetModule`, and broadcast to all other relevant clients to maintain a consistent state.
*   **Concurrency Control:** Implements a strict locking mechanism (`globalEditLock`) to serialize filesystem and in-memory asset modifications, preventing race conditions between multiple editors.
*   **Asset Abstraction:** Works with an `AssetTypeRegistry` to handle different kinds of assets (e.g., JSON, Textures, Models) polymorphically, delegating validation and processing logic to specialized `AssetTypeHandler` implementations.

### Lifecycle & Ownership
- **Creation:** The AssetEditorPlugin is instantiated once by the server's `PluginManager` during the server bootstrap sequence. The static singleton instance is set within its `setup` method, making it globally accessible via `AssetEditorPlugin.get()`.
- **Scope:** The plugin is a long-lived singleton. Its lifetime is tied directly to the server's runtime; it persists as long as the plugin is enabled.
- **Destruction:** The `shutdown` method is invoked by the `PluginManager` when the server is shutting down or the plugin is being disabled. This process involves gracefully disconnecting all connected `EditorClient`s, canceling scheduled tasks like client pings, and shutting down all managed `DataSource` instances.

## Internal State & Concurrency
- **State:** The AssetEditorPlugin is highly stateful and mutable. It maintains several critical in-memory data structures:
    - A map of all connected clients (`uuidToEditorClients`).
    - A mapping of which asset each client currently has open (`clientOpenAssetPathMapping`).
    - A collection of `DataSource` objects, representing the loaded asset packs.
    - A cache of loaded asset `Schema` definitions.
    - An `UndoRedoManager` to track modification history for clients.

- **Thread Safety:** This class is **not** inherently thread-safe and must be accessed with extreme caution. Concurrency is managed internally through a combination of strategies:
    - **Global Write Lock:** A `StampedLock` named `globalEditLock` is used to serialize all critical write operations (e.g., creating, updating, deleting assets on disk). Any operation that modifies the filesystem or core asset state must acquire this lock to prevent data corruption.
    - **Initialization Lock:** A separate `StampedLock` named `initLock` manages the asynchronous initialization process, safely queuing clients that connect while the plugin is still loading schemas and building asset trees.
    - **Concurrent Collections:** Utilizes `ConcurrentHashMap` for client and data source collections to allow for safe reads and additions/removals without blocking the entire plugin.

**WARNING:** External systems interacting with the AssetEditorPlugin must not assume thread safety. Direct modification of its internal state without acquiring the appropriate locks will lead to system instability and data loss.

## API Surface
The public API is dominated by `handle...` methods, which are designed to be invoked by network packet handlers in response to client requests.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | AssetEditorPlugin | O(1) | Static accessor for the singleton instance. |
| handleInitializeClient(EditorClient) | void | O(N) | Manages the complex initialization sequence for a new client, including state checks and packet dispatch. |
| handleCreateAsset(client, path, data, ...) | void | O(N) | Handles a request to create a new asset. Involves I/O, locking, and broadcasting updates. |
| handleJsonAssetUpdate(client, path, commands, ...) | void | O(N) | Processes a set of structured JSON patch commands against an asset. Acquires the global lock. |
| handleUpdateAssetPack(client, packId, manifest) | void | O(N) | Updates the manifest for an asset pack, writes it to disk, and re-registers the pack if its identifier changed. |
| handleDeleteAsset(client, path, ...) | void | O(N) | Handles a request to delete an asset. Involves I/O, lock acquisition, and broadcasting updates. |

## Integration Patterns

### Standard Usage
The AssetEditorPlugin is a self-contained system driven by network events. Direct interaction is uncommon. Other server plugins or systems should retrieve the singleton instance to access its state or services.

```java
// Example: A custom admin command checking editor status
AssetEditorPlugin editor = AssetEditorPlugin.get();
Collection<DataSource> sources = editor.getDataSources();
System.out.println("Currently loaded data sources: " + sources.size());

// Note: Accessing client maps or other mutable state would require
// careful consideration of the internal locking mechanisms.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new AssetEditorPlugin()`. The plugin is managed by the server's plugin framework. Always use the static `get()` method.
- **Bypassing Locks:** Do not attempt to modify asset files on disk directly while the server is running. All modifications must go through the plugin's API to ensure the `globalEditLock` is respected and state is synchronized correctly.
- **Ignoring Initialization State:** Do not assume the plugin is ready immediately on server start. The initialization process is asynchronous. Attempting to access schemas or asset trees before the `InitState` is `INITIALIZED` will result in errors.

## Data Pipeline
The flow of data for a typical asset modification is a multi-stage process involving the client, plugin, filesystem, and game runtime.

> Flow:
> EditorClient Packet (e.g., AssetEditorUpdateAsset) -> Server Packet Handler -> **AssetEditorPlugin**.handleAssetUpdate -> Acquire `globalEditLock` -> `DataSource`.updateAsset -> Filesystem Write -> `AssetModule` Reload -> Game Runtime Update -> Broadcast `AssetEditorAssetUpdated` to other clients

