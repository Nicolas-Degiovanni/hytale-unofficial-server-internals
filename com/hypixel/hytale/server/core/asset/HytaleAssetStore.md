---
description: Architectural reference for HytaleAssetStore
---

# HytaleAssetStore

**Package:** com.hypixel.hytale.server.core.asset
**Type:** Transient

## Definition
```java
// Signature
public class HytaleAssetStore<K, T extends JsonAssetWithMap<K, M>, M extends AssetMap<K, T>> extends AssetStore<K, T, M> {
```

## Architecture & Concepts
The HytaleAssetStore is the server-side, network-aware implementation of the generic AssetStore. Its primary architectural role is to manage the lifecycle of a specific category of game assets (e.g., items, prefabs, block definitions) and ensure their state is synchronized between the server's memory and all connected game clients.

It acts as a critical bridge between three core systems:
1.  **File System:** It loads asset definitions from JSON files on disk.
2.  **Game State:** It maintains the canonical, in-memory collection of all loaded assets of its type within an AssetMap.
3.  **Network Layer:** It serializes asset data into network packets to broadcast initial state, updates, and removals to clients.

A key feature is its integration with the AssetMonitor service, which enables **live reloading** of assets. When a developer modifies an asset file on disk while the server is running, HytaleAssetStore detects the change, reloads the asset, and automatically pushes the update to all clients, facilitating rapid iteration without server restarts.

## Lifecycle & Ownership
-   **Creation:** An instance is created exclusively through its fluent Builder. This construction is typically performed once per asset type during the server's bootstrap sequence, orchestrated by the AssetModule. The Builder configures essential strategies, such as the AssetPacketGenerator responsible for network serialization.
-   **Scope:** An instance persists for the entire server session. It is a long-lived, stateful component that holds the ground truth for its managed asset type.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down and the AssetModule releases its references. The removeFileMonitor method provides a mechanism for explicit teardown, detaching it from the file monitoring service.

## Internal State & Concurrency
-   **State:** HytaleAssetStore is highly mutable. Its core state is the assetMap inherited from the parent AssetStore, which contains all loaded asset data. Additionally, it maintains a `cachedInitPackets` field, wrapped in a SoftReference. This is a critical performance optimization that caches the large initial synchronization packet sent to new clients. This cache is aggressively invalidated (set to null) whenever any asset is updated or removed, ensuring data consistency.

-   **Thread Safety:** This class is **not thread-safe** and is designed to be accessed only from the main server thread. Methods that modify the internal assetMap or invalidate the packet cache, such as handleRemoveOrUpdate, are not protected by locks. The internal AssetStoreMonitorHandler processes file system events and invokes modification methods on its parent HytaleAssetStore. The underlying AssetMonitor system is therefore required to dispatch these file change events synchronously on the main server thread to prevent race conditions and state corruption.

    **WARNING:** Accessing or modifying a HytaleAssetStore instance from any thread other than the main server thread will lead to unpredictable behavior, including corrupted asset state, client desynchronization, and server crashes.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addFileMonitor(packKey, assetsPath) | void | O(1) | Registers a directory with the global AssetMonitor for live-reloading. Idempotent. |
| removeFileMonitor(path) | void | O(1) | Deregisters a directory from the AssetMonitor. |
| handleRemoveOrUpdate(toBeRemoved, toBeUpdated, query) | void | O(P) | Generates and broadcasts update/remove packets to all P connected players. Invalidates the init packet cache. |
| sendAssets(packetConsumer) | void | O(1) or O(A) | Sends all assets to a consumer. Complexity is O(1) on a cache hit, or O(A) where A is the total number of assets on a cache miss. |
| builder(kClass, tClass, assetMap) | Builder | O(1) | Static factory method to construct a new Builder for this store. |

## Integration Patterns

### Standard Usage
A HytaleAssetStore is configured and built during server initialization. It is not intended for direct, frequent interaction by general game logic. Other systems query the asset data via the store's underlying AssetMap.

```java
// Example setup within an AssetModule or similar bootstrap class
AssetMap<String, ItemAsset, ItemAssetMap> itemAssetMap = new ItemAssetMap();

HytaleAssetStore<String, ItemAsset, ItemAssetMap> itemStore = HytaleAssetStore
    .builder(ItemAsset.class, itemAssetMap)
    .setLogger(serverLogger)
    .setPacketGenerator(new ItemAssetPacketGenerator()) // CRITICAL for client sync
    .setNotificationItemFunction(key -> createItemNotification(key))
    .build();

// Register the store for live-reloading
itemStore.addFileMonitor("core", server.getPath("assets/content/items"));

// Load initial assets
itemStore.loadAssetsFromDirectory("core", server.getPath("assets/content/items"));
```

### Anti-Patterns (Do NOT do this)
-   **Manual Instantiation:** Do not attempt to instantiate directly. The constructor is protected. Always use the static `builder` method to ensure proper initialization.
-   **Omitting Packet Generator:** Building a store without calling `setPacketGenerator` will render it incapable of synchronizing assets with clients. The server will have the assets, but clients will not.
-   **Concurrent Modification:** Do not call methods like `loadAssetsFromPaths` or `removeAssetWithPaths` from a background thread. All modifications must be sequenced on the main server thread.

## Data Pipeline
The HytaleAssetStore manages two primary data flows: live asset reloading and initial client synchronization.

**Live Reload Pipeline:**
> File System Event (Create/Modify/Delete) -> AssetMonitor -> `AssetStoreMonitorHandler` -> **HytaleAssetStore**.load/remove -> **HytaleAssetStore**.handleRemoveOrUpdate -> AssetPacketGenerator -> Universe.broadcastPacketNoCache -> Network Layer -> All Connected Clients

**New Client Synchronization Pipeline:**
> New Client Joins -> Server Connection Logic -> **HytaleAssetStore**.sendAssets -> (Cache Miss) -> AssetPacketGenerator.generateInitPacket -> (Cache Write) -> Network Packet -> Individual Client

