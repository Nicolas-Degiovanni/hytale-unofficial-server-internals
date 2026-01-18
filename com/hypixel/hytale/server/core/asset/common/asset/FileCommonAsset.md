---
description: Architectural reference for FileCommonAsset
---

# FileCommonAsset

**Package:** com.hypixel.hytale.server.core.asset.common.asset
**Type:** Transient Model

## Definition
```java
// Signature
public class FileCommonAsset extends CommonAsset {
```

## Architecture & Concepts
The FileCommonAsset class is a specialized implementation of the CommonAsset abstraction. It represents a game asset whose canonical data source is a file on the physical filesystem. Its primary architectural role is to act as a lazy, asynchronous proxy to the asset's binary data.

Instead of loading the file's contents into memory upon instantiation, a FileCommonAsset holds only a Path object pointing to the file's location. The actual file I/O is deferred until the `getBlob0` method is invoked. This operation is performed asynchronously on a background thread, returning a CompletableFuture. This design is critical for performance, as it prevents the main application threads from blocking on expensive disk I/O operations, especially during server startup or when loading large numbers of assets.

This class effectively decouples the *existence* of an asset from its *in-memory representation*, allowing the engine to manage a registry of thousands of assets without consuming memory for assets that are not immediately needed.

### Lifecycle & Ownership
- **Creation:** Instances are created by higher-level asset management systems, such as an AssetScanner or AssetRepository, during the discovery phase. A new FileCommonAsset is typically instantiated for each asset file found in the server's asset directories. Direct instantiation by game logic is strongly discouraged.
- **Scope:** The lifetime of a FileCommonAsset object is tied to the asset registry that contains it. It persists as long as the asset is considered known to the system. This is typically the entire duration of the server session.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection once it is removed from the asset registry and no other systems hold a reference to it. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The internal state, specifically the Path to the asset file, is **immutable**. It is set once via the constructor in a final field and cannot be changed. The object itself is a handle to external state (the file on disk), which may be mutable.
- **Thread Safety:** This class is **conditionally thread-safe**. Accessing its immutable state via `getFile` is always safe. The primary method, `getBlob0`, is designed for concurrent execution. It safely offloads the I/O operation to the common ForkJoinPool or a similar executor, ensuring that multiple threads can request the asset's data simultaneously without causing race conditions within the object itself.

**WARNING:** While the object is thread-safe, the underlying file is subject to external filesystem operations. A call to `getBlob0` may fail if the file is deleted or modified by another process between the time of instantiation and the read operation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFile() | Path | O(1) | Returns the immutable filesystem path for this asset. |
| getBlob0() | CompletableFuture<byte[]> | O(N) | Asynchronously reads the asset's complete binary content from disk. N is the size of the file. This is a non-blocking, I/O-bound operation. The returned future will complete exceptionally if the file cannot be read. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve an instance from a central asset manager and then asynchronously request its data, chaining subsequent processing steps to the returned future.

```java
// Asset instances should always be retrieved from a central service.
AssetManager assetManager = context.getService(AssetManager.class);
FileCommonAsset modelAsset = (FileCommonAsset) assetManager.getAsset("prefabs/golem.hmdl");

// Asynchronously request the data and process it upon completion.
// This does NOT block the current thread.
modelAsset.getBlob0().thenAccept(modelBytes -> {
    // This logic executes on a worker thread once the file is read.
    ModelLoader.load(modelBytes);
}).exceptionally(error -> {
    // Always handle potential I/O errors.
    log.error("Failed to load asset: " + modelAsset.getFile(), error);
    return null;
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FileCommonAsset()`. The asset system must be the sole authority for creating and managing asset handles to ensure registry integrity and prevent duplicate objects.
- **Blocking on Futures:** Calling `get()` on the CompletableFuture returned by `getBlob0` from a performance-critical thread (e.g., the main server tick loop) is a severe anti-pattern. It nullifies the entire benefit of the asynchronous design and will cause the thread to block on disk I/O, leading to stutter and poor performance.
- **Ignoring I/O Failures:** The `getBlob0` operation can easily fail if the asset file is missing, corrupted, or locked. Failing to attach an `exceptionally` handler to the CompletableFuture will result in silent failures or uncaught exceptions that can terminate the worker thread.

## Data Pipeline
FileCommonAsset acts as a data source node in the asset processing pipeline. It is responsible for initiating the flow of data from the filesystem into the server's in-memory systems.

> Flow:
> Filesystem Discovery -> **FileCommonAsset** (Instantiation as a handle) -> Asynchronous `getBlob0()` call -> Raw byte[] stream -> Asset Consumer (e.g., ModelLoader, TextureDecoder)

