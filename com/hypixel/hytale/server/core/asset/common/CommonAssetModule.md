---
description: Architectural reference for CommonAssetModule
---

# CommonAssetModule

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Singleton (Plugin)

## Definition
```java
// Signature
public class CommonAssetModule extends JavaPlugin {
```

## Architecture & Concepts
The CommonAssetModule is a core server plugin that orchestrates the entire lifecycle of shared, or "common", assets. It acts as the authoritative source for assets that must be synchronized from the server to all connected clients. This module is the critical bridge between the server's persistent storage (represented by AssetPacks on the file system) and the in-memory state of connected game clients.

Its responsibilities are threefold:
1.  **Discovery & Loading:** During server startup and when new AssetPacks are registered, it scans directories, reads optimized index and cache files (CommonAssetsIndex.hashes, CommonAssetsIndex.cache), and loads asset metadata into the central CommonAssetRegistry. For assets not found in caches, it reads the file bytes from disk asynchronously.
2.  **Synchronization:** It serializes asset data into a sequence of network packets (AssetInitialize, AssetPart, AssetFinalize) and broadcasts them to clients. This occurs when a client first connects, when an asset is modified on the server, or when an asset is added or removed.
3.  **Live Reloading:** For mutable (non-production) AssetPacks, it integrates with the AssetMonitor service to listen for real-time file system changes. It handles file creation, modification, and deletion events, automatically reloading the corresponding assets and pushing updates to all clients, enabling hot-reloading for developers.

This module is designed as a Hytale JavaPlugin, meaning its lifecycle is managed by the server's plugin system. It hooks into the server's event bus to react to major system events like `AssetPackRegisterEvent` and `LoadAssetEvent`.

## Lifecycle & Ownership
-   **Creation:** A single instance is created by the server's plugin loader during the initial bootstrap sequence. The constructor receives a JavaPluginInit object, providing it with necessary server context. The static `instance` field is set within the constructor.
-   **Scope:** The instance is global and persists for the entire lifetime of the server process. It is accessible via the static `get()` method.
-   **Destruction:** The module is unloaded and its resources are released when the HytaleServer instance shuts down. There is no explicit public destruction method; cleanup is managed by the plugin framework.

## Internal State & Concurrency
-   **State:** The CommonAssetModule is highly stateful and mutable. Its primary state is managed externally in the static `CommonAssetRegistry`, which holds the canonical list of all loaded common assets. It also maintains an internal `assets` field, a `CachedSupplier` that provides a memoized array of asset packets. This cache is explicitly invalidated whenever the underlying registry changes via `assets.invalidate()`.

-   **Thread Safety:** This class is **not thread-safe**. While it heavily utilizes `CompletableFuture` to perform file I/O operations asynchronously and off the main server thread, the modification of the shared `CommonAssetRegistry` and its own internal cache is not synchronized. The design implicitly relies on a single-threaded execution model for state mutation, likely by ensuring that event handlers and modification logic are executed on the main server tick thread.

    **WARNING:** Calling methods that modify state, such as `addCommonAsset` or `removeCommonAssets`, from multiple threads concurrently will lead to race conditions, registry corruption, and unpredictable client behavior. All interactions that mutate asset state must be synchronized or dispatched to the main server thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | CommonAssetModule | O(1) | Returns the global singleton instance. |
| loadCommonAssets(pack, bootTime) | void | O(N * M) | Scans an AssetPack for common assets. Complexity depends on the number of files (N) and their size (M). This is a blocking, high-latency operation. |
| addCommonAsset(pack, asset, log) | void | O(log N) | Registers a new asset or updates an existing one in the registry. Invalidates internal caches and broadcasts updates to clients. |
| getRequiredAssets() | Asset[] | O(1) / O(N) | Returns a cached array of all assets required by clients. The first call is O(N) to build the cache. |
| sendAssetsToPlayer(handler, assets, force) | void | O(N * M) | Streams a list of assets to a specific client. A blocking network operation. |
| sendRemoveAssets(assets, force) | void | O(N) | Broadcasts a packet to all clients instructing them to remove the specified assets. |

## Integration Patterns

### Standard Usage
Interaction with this module is primarily event-driven. Other systems trigger asset loading by firing events that the module is registered to listen for.

```java
// This is typically done by the core server, not user code.
// An event is fired to trigger the loading process for a given asset pack.
IEventDispatcher dispatcher = HytaleServer.get().getEventBus();
dispatcher.dispatch(new AssetPackRegisterEvent(someAssetPack));

// The CommonAssetModule listens for this event and automatically calls
// its internal loadCommonAssets method.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new CommonAssetModule()`. The server's plugin framework is solely responsible for its creation. Always use `CommonAssetModule.get()` to retrieve the singleton instance.
-   **Concurrent Modification:** Do not call `addCommonAsset` or other state-mutating methods from an uncontrolled, asynchronous context. This will corrupt the `CommonAssetRegistry`. All modifications should be funneled through the server's main thread or a dedicated, synchronized job queue.
-   **Ignoring Cache Invalidations:** Reading from the `assets` supplier can yield stale data if the registry has been modified and the cache has not been invalidated. Methods like `addCommonAsset` handle this internally, but manual manipulation of the registry will break this contract.

## Data Pipeline

The module processes data through two primary pipelines: asset loading and asset synchronization.

**Pipeline 1: Live Asset Reload**
> Flow:
> File System Event (Create/Modify/Delete) -> AssetMonitor -> **CommonAssetModule.CommonAssetMonitorHandler** -> `reloadAsset()` or `CommonAssetRegistry.removeCommonAssetByName()` -> `sendAsset()` or `sendRemoveAssets()` -> Universe Packet Broadcast -> Network Layer -> All Clients

**Pipeline 2: New Player Connection**
> Flow:
> Player Joins -> `SendCommonAssetsEvent` Fired -> **CommonAssetModule.onSendCommonAssets()** -> `sendAssetsToPlayer()` -> Asset Data chunked into `AssetInitialize`, `AssetPart`, `AssetFinalize` packets -> PacketHandler -> Network Layer -> Single Client

