---
description: Architectural reference for DiskResourceStorageProvider
---

# DiskResourceStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.world.storage.resources
**Type:** Factory / Provider

## Definition
```java
// Signature
public class DiskResourceStorageProvider implements IResourceStorageProvider {
```

## Architecture & Concepts
The DiskResourceStorageProvider is a factory component responsible for creating instances of `IResourceStorage` that persist game data to the local filesystem. It acts as a bridge between the abstract concept of resource storage and a concrete, file-based implementation.

This class is a key part of the server's data persistence strategy. It is designed to be a pluggable provider, identified by its static `ID` field "Disk". The server's configuration can specify this provider to instruct a world to save its resources as individual JSON files on disk.

The provider itself is lightweight; its primary role is to hold the configuration for the storage path (e.g., "resources") and to instantiate the `DiskResourceStorage` worker class, which performs the actual I/O operations. The presence of a `CODEC` indicates that this provider is configured and managed declaratively, typically within a world's master configuration file.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the Hytale `Codec` system during the deserialization of a world's configuration. A higher-level service, such as a `WorldManager`, reads the configuration and uses the `CODEC` to create the provider instance.
-   **Scope:** The provider's lifecycle is tied to the loaded world configuration. It persists as long as the server has the world's settings in memory, not necessarily for the duration of a running world session.
-   **Destruction:** The object is eligible for garbage collection when the world configuration it belongs to is unloaded. It has no native resources to release.

## Internal State & Concurrency
-   **State:** The internal state is minimal and effectively immutable. It consists of a single `path` string, which is set during deserialization via the `CODEC`. This path is relative to the world's save directory.
-   **Thread Safety:** This class is thread-safe. Its state is immutable after construction, and its primary method, `getResourceStorage`, is re-entrant, creating a new, independent `DiskResourceStorage` object on each invocation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceStorage(World world) | IResourceStorage | O(1) | Factory method. Creates and returns a new `DiskResourceStorage` instance configured for the specified world's save path. |
| getPath() | String | O(1) | Returns the configured relative path for resource storage. |
| migrateFiles(World world) | static void | O(N) | **WARNING:** A destructive utility to migrate data from legacy `chunkstore` and `entitystore` directories. This should only be used for server maintenance. |

## Integration Patterns

### Standard Usage
This provider is not intended for direct use by game logic developers. It is configured at the world level. The engine's persistence layer uses this provider to obtain a storage implementation for a given world.

A conceptual world configuration might look like this:
```json
{
  "storage": {
    "resources": {
      "type": "Disk",
      "Path": "world_data/resources"
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new DiskResourceStorageProvider()`. This bypasses the configuration system and will always use the default path "resources". The provider must be instantiated via its `CODEC` from a configuration source.
-   **Improper Migration:** Do not call the static `migrateFiles` method on a live, running world. This is a maintenance task that should be performed on an offline server with proper backups, as it involves moving and deleting directories.

---
# DiskResourceStorageProvider.DiskResourceStorage

**Package:** com.hypixel.hytale.server.core.universe.world.storage.resources
**Type:** Transient

## Definition
```java
// Signature
public static class DiskResourceStorage implements IResourceStorage {
```

## Architecture & Concepts
DiskResourceStorage is the concrete implementation of the `IResourceStorage` interface. It performs the low-level I/O operations required to load, save, and delete game resources from the filesystem. Each instance is bound to a specific, absolute directory path provided by its parent `DiskResourceStorageProvider`.

The core responsibility of this class is to manage the serialization and deserialization of `Resource` objects. It uses the Hytale `Codec` system to convert in-memory Java objects into BSON documents, which are then written to disk as human-readable JSON files. The filename for each resource is derived from its unique resource ID, with a ".json" extension.

Error handling is a key aspect of its design. When loading a resource, it gracefully handles missing files, empty files, or parsing errors by logging a warning and returning a new, default-constructed `Resource` instance. This ensures that data corruption in one resource file does not prevent the server from starting.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by `DiskResourceStorageProvider.getResourceStorage`. It is created on-demand when a system needs to interact with a world's persisted resources.
-   **Scope:** The object is typically short-lived. A service will request it, perform one or more I/O operations, and then release its reference. It is not designed to be a long-lived, stateful service.
-   **Destruction:** The object is garbage collected when it is no longer referenced. It holds no system resources that require explicit cleanup.

## Internal State & Concurrency
-   **State:** The only state is a final `Path` object representing the absolute path to the resources directory. This state is immutable.
-   **Thread Safety:** This class is thread-safe for concurrent, independent operations. All public methods (`load`, `save`, `remove`) return a `CompletableFuture` and delegate their blocking I/O work to a background thread pool via `CompletableFuture.supplyAsync` or similar mechanisms.

**WARNING:** While the class methods are safe to call from multiple threads, there is no built-in file-locking mechanism. Initiating two concurrent `save` operations for the *same resource ID* will create a race condition at the filesystem level, potentially leading to a corrupted file. Higher-level application logic is responsible for ensuring that a given resource is not saved simultaneously by multiple threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load(...) | CompletableFuture<T> | O(S) | Asynchronously reads a JSON file from disk, deserializes it into a `Resource` object, and returns it in a future. S is the size of the file. |
| save(...) | CompletableFuture<Void> | O(S) | Asynchronously serializes a `Resource` object into a BSON document and writes it to a JSON file, overwriting any existing file. S is the size of the object. |
| remove(...) | CompletableFuture<Void> | O(1) | Asynchronously deletes the JSON file corresponding to the given resource type. Fails silently if the file does not exist. |

## Integration Patterns

### Standard Usage
This class is typically used by engine-level systems that manage the persistence of ECS components. A developer would interact with a higher-level `Store` or `World` API, which in turn delegates the persistence operations to an instance of `DiskResourceStorage`.

```java
// Engine-level code (conceptual)
IResourceStorage storage = provider.getResourceStorage(currentWorld);

// Save a resource
CompletableFuture<Void> saveFuture = storage.save(store, data, resourceType, resourceInstance);
saveFuture.thenRun(() -> System.out.println("Save complete."));

// Load a resource
CompletableFuture<MyResource> loadFuture = storage.load(store, data, resourceType);
loadFuture.thenAccept(resource -> processResource(resource));
```

### Anti-Patterns (Do NOT do this)
-   **Blocking on Futures:** Do not call `.get()` or `.join()` on the returned `CompletableFuture` from a main game loop or network thread. This will block the thread, causing severe performance degradation and server lag. Use asynchronous callbacks like `thenAccept` or `thenRun`.
-   **Ignoring Failures:** Do not ignore potential exceptions from the `CompletableFuture`. While the `load` method is resilient, `save` and `remove` can fail due to I/O exceptions (e.g., disk full, permissions errors). Chain exception handling with `.exceptionally()`.
-   **Frequent Small Writes:** Avoid saving a resource after every minor change. This can cause excessive disk I/O. Higher-level systems should implement strategies like periodic saving or "dirty" flags to batch writes.

## Data Pipeline
This class is the endpoint for the resource persistence pipeline. It handles the final translation between in-memory objects and their on-disk representation.

> **Save Flow:**
> In-Memory `Resource` Object -> `BuilderCodec` Serialization -> `BsonDocument` -> **DiskResourceStorage.save** -> `BsonUtil` Writer -> JSON File on Disk

> **Load Flow:**
> JSON File on Disk -> **DiskResourceStorage.load** -> `RawJsonReader` -> `BuilderCodec` Deserialization -> In-Memory `Resource` Object -> `CompletableFuture` Wrapper

