---
description: Architectural reference for CommonAsset
---

# CommonAsset

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Abstract Data Model

## Definition
```java
// Signature
public abstract class CommonAsset implements NetworkSerializable<Asset> {
```

## Architecture & Concepts
The CommonAsset class is the abstract foundation for all server-side game assets. It serves as a metadata container and a proxy for an asset's binary data, rather than the data itself. This design is critical for server performance and memory management.

The core architectural principles are:

1.  **Identity vs. Data Separation:** An instance of CommonAsset represents an asset's identity (its unique *name* and content-derived *hash*). The actual binary data (the *blob*) is decoupled and loaded on demand.
2.  **Asynchronous Lazy Loading:** The asset's binary content is not loaded into memory upon instantiation. It is fetched asynchronously via the getBlob method, which returns a CompletableFuture. This prevents blocking server threads on I/O operations, such as reading from disk or a remote store. The abstract getBlob0 method implements the Template Method Pattern, forcing subclasses to provide a concrete loading strategy.
3.  **Aggressive Memory Caching:** The class employs a multi-level caching strategy using Java's reference types to minimize memory footprint:
    *   **WeakReference (for blob):** The reference to the CompletableFuture containing the raw byte array is weak. This allows the garbage collector to reclaim the potentially large byte array if no other part of the system holds a strong reference to it, even if the CommonAsset object itself is still referenced.
    *   **SoftReference (for packet):** The network-serializable Asset packet is held by a soft reference. This object will be retained as long as memory is not under pressure, providing a fast path for network serialization, but can be reclaimed by the JVM if needed.

This architecture ensures that the server can manage references to millions of assets without necessarily holding all their data in memory simultaneously.

### Lifecycle & Ownership
-   **Creation:** Instances of CommonAsset subclasses are created by a central asset management service (e.g., an AssetManager) during server bootstrap or in response to dynamic asset discovery. Direct instantiation by game logic is discouraged.
-   **Scope:** The lifetime of a CommonAsset object is typically tied to the server's master asset registry. It persists as long as the asset is considered available to the server.
-   **Destruction:** The object is eligible for garbage collection when it is removed from the asset registry and no other services hold a reference. Critically, its internal caches for the blob and network packet can be reclaimed by the garbage collector *before* the CommonAsset object itself, responding dynamically to memory pressure.

## Internal State & Concurrency
-   **State:** The object's identity fields, name and hash, are immutable and set at construction. Its internal state consists of transient, mutable caches (`blob`, `cachedPacket`) that are populated on first access.
-   **Thread Safety:** **This class is not thread-safe.** The lazy initialization logic in the getBlob method contains a check-then-act race condition. If multiple threads call getBlob concurrently on a cold object, the abstract getBlob0 method may be invoked multiple times.

    **WARNING:** All access to and modification of a CommonAsset instance, particularly the initial call to getBlob, must be externally synchronized by the calling context, such as the AssetManager. Failure to do so can result in redundant I/O operations and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getName() | String | O(1) | Returns the canonical, forward-slash-separated name of the asset. |
| getHash() | String | O(1) | Returns the lowercase SHA-256 hash of the asset's content. |
| getBlob() | CompletableFuture<byte[]> | O(1) / O(N) | Asynchronously retrieves the asset's binary data. O(1) if cached, O(N) on first call, where N is the cost of the I/O operation. |
| toPacket() | Asset | O(1) | Constructs a lightweight, network-serializable representation of the asset. This operation is cached. |

## Integration Patterns

### Standard Usage
The intended use is through a service locator or dependency injection pattern, where a central service manages the asset lifecycle. Game logic requests an asset's metadata and then asynchronously requests its data when needed.

```java
// AssetService is a hypothetical manager class
AssetService assetService = serverContext.getAssetService();

// 1. Retrieve the asset metadata object. This is a cheap operation.
CommonAsset modelAsset = assetService.getAsset("models/character/t_rex.hymodel");

// 2. Asynchronously request the binary data when required for processing.
modelAsset.getBlob().thenAccept(bytes -> {
    // This code executes on a worker thread when the data is loaded.
    ModelData model = ModelParser.parse(bytes);
    world.spawnEntity(model);
}).exceptionally(error -> {
    // Handle I/O errors, e.g., file not found.
    log.error("Failed to load asset blob: " + modelAsset.getName(), error);
    return null;
});
```

### Anti-Patterns (Do NOT do this)
-   **Holding Strong Blob References:** Do not store the `byte[]` array returned from `getBlob()` in a long-lived field. This defeats the weak reference caching mechanism and can lead to excessive memory consumption. Re-request the blob when you need it.
-   **Concurrent Initialization:** Do not call `getBlob()` from multiple threads without external locking. This will lead to race conditions as described in the Concurrency section.
-   **Manual Hashing:** Do not manually hash asset bytes to find an asset. Rely on the `AssetService` to manage the mapping between asset names and their corresponding CommonAsset instances.

## Data Pipeline
The CommonAsset class acts as a crucial node in the server's data flow, bridging asset storage with the network and game logic layers.

> **Flow (Asset Loading & Processing):**
> Filesystem / Database -> AssetManager discovers and creates **CommonAsset** instance -> Game Logic requests asset by name -> **CommonAsset**.getBlob() -> Asynchronous I/O to load data -> Game Logic processes bytes

> **Flow (Network Transmission):**
> Client requests asset -> Network Handler asks AssetManager for asset -> **CommonAsset**.toPacket() -> Network Layer serializes and sends `Asset` packet -> Client receives packet

