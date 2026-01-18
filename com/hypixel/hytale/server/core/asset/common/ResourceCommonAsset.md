---
description: Architectural reference for ResourceCommonAsset
---

# ResourceCommonAsset

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Transient

## Definition
```java
// Signature
public class ResourceCommonAsset extends CommonAsset {
```

## Architecture & Concepts
ResourceCommonAsset is a concrete implementation of the CommonAsset abstraction. Its primary architectural role is to represent an asset whose source data is embedded within the application's classpath, typically inside a JAR file. It acts as a bridge between the low-level Java ClassLoader resource API and the high-level Hytale Asset Management system.

Unlike other asset types that might load from a filesystem or network, this class is purpose-built for hermetically sealed, pre-packaged assets. The identity of a resource is defined by a combination of a reference Class and a relative path, which the Java runtime resolves into an InputStream.

The class encapsulates the I/O logic for reading these embedded resources, presenting the resulting byte array through the asynchronous contract defined by its parent, CommonAsset.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created via the static factory method `ResourceCommonAsset.of`. This method is typically invoked during server or client bootstrap by an asset scanning service that enumerates known resource paths. Direct instantiation by game logic is heavily discouraged.

- **Scope:** A ResourceCommonAsset is a short-lived object. Its lifetime is generally confined to the scope of the asset registration process. Once its data is loaded and potentially cached by a higher-level manager (like an AssetManager), the ResourceCommonAsset instance itself is no longer needed and becomes eligible for garbage collection.

- **Destruction:** There is no explicit destruction or cleanup method. The underlying `InputStream` used for reading resource data is managed internally by try-with-resources blocks within the `of` and `getBlob0` methods, ensuring that file handles are closed immediately after the byte data is read into memory.

## Internal State & Concurrency
- **State:** The object's state, consisting of the reference class, path, name, and hash, is **immutable**. All fields are set at construction time and cannot be changed. While the parent CommonAsset may cache the byte array, this specific implementation re-reads the data from the classpath on each call to `getBlob0`. The parent class is responsible for memoizing the result of this operation.

- **Thread Safety:** This class is **thread-safe**. Its immutable state and re-entrant data loading method (`getBlob0`) allow it to be safely accessed and used by multiple threads simultaneously without external locking. The use of CompletableFuture in its API contract further reinforces its suitability for asynchronous, multi-threaded asset loading pipelines.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(clazz, path, name) | ResourceCommonAsset | O(N) | **Primary Factory.** Creates an instance by reading a classpath resource. Returns null if the resource is not found. Throws an unchecked exception on I/O failure. |
| getBlob0() | CompletableFuture<byte[]> | O(N) | **Asynchronous Data Access.** Fulfills the CommonAsset contract by reading the resource bytes from the classpath. Returns a failed future on I/O error. |
| getPath() | String | O(1) | Returns the raw resource path used for loading. |

*Complexity N refers to the size of the asset file in bytes.*

## Integration Patterns

### Standard Usage
The intended use is by asset discovery systems that scan the classpath and register discovered assets with a central manager. The `of` method provides a convenient, single-call mechanism for instantiation.

```java
// Example from an asset registration service
// Note: HytaleServer.class is used as an anchor to find the resource
ResourceCommonAsset logoAsset = ResourceCommonAsset.of(
    HytaleServer.class, 
    "/assets/hytale/textures/gui/logo.png", 
    "hytale:textures/gui/logo"
);

if (logoAsset != null) {
    assetRegistry.register(logoAsset);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ResourceCommonAsset()`. The constructor requires you to pre-load the byte array, which defeats the purpose of the class's internal loading logic. The static `of` method correctly handles cases where a resource might not exist.

- **Incorrect Pathing:** The resource path is resolved relative to the classpath structure. Providing an absolute filesystem path or an incorrect relative path will cause `clazz.getResourceAsStream` to fail, resulting in a null return from the `of` method.

- **Blocking on Futures:** The `getBlob0` method returns a CompletableFuture. Blocking the main game or server thread to wait for this future to complete is a severe performance anti-pattern. Always chain dependent logic using asynchronous callbacks.

## Data Pipeline
This class acts as a data source node in the broader asset pipeline. It is responsible for the initial ingestion of asset data from the application package into memory.

> Flow:
> Classpath Resource (in JAR) -> `InputStream` -> **ResourceCommonAsset** -> `CompletableFuture<byte[]>` -> AssetManager Cache -> Game System (e.g., Renderer)

