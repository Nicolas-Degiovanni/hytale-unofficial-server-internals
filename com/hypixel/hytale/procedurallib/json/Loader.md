---
description: Architectural reference for Loader
---

# Loader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class Loader<K extends SeedResource, T> {
```

## Architecture & Concepts
The Loader class is an abstract base class that establishes a foundational pattern for loading file-based resources within the procedural content system. It serves as a contract for all concrete loader implementations, decoupling the request for a resource from the specific mechanics of its retrieval and deserialization.

Its primary role is to define a common interface for operations that take a resource identifier, a **SeedString**, and a root directory, the **dataFolder**, and produce a fully realized in-memory object of type **T**. The use of generics for both the key type **K** and the return type **T** makes this a highly flexible and reusable component for loading diverse asset types, such as JSON objects, models, or textures, from the filesystem.

This class embodies the Command design pattern, where each instance is a self-contained request for a single loading operation.

### Lifecycle & Ownership
- **Creation:** As an abstract class, Loader is never instantiated directly. Concrete subclasses are created on-demand by higher-level systems, such as an AssetManager or a ProceduralContentRegistry, whenever a specific resource needs to be loaded from disk. The constructor is invoked with the context required for the load: the resource identifier and the base path.
- **Scope:** Instances of Loader are transient and short-lived. They are designed to exist only for the duration of a single call to the load method. They are not services and should not be cached or reused.
- **Destruction:** The object is eligible for garbage collection as soon as the load method completes and the caller no longer holds a reference. It manages no persistent resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state, consisting of the seed and dataFolder, is effectively immutable. It is set once during construction and is not intended to be modified. The Loader itself does not cache the result of the load operation; each call to load is expected to re-read from the source.
- **Thread Safety:** The base Loader class is thread-safe due to its immutable state. However, the responsibility for thread safety in the loading process lies entirely with the concrete subclasses. Implementations of the load method that perform file I/O are inherently blocking operations.

    **WARNING:** Calling load on a performance-critical thread, such as the main game loop or rendering thread, will cause severe application stalls. All loading operations should be dispatched to a dedicated worker thread pool or an asynchronous task system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | T | I/O Bound | Abstract method to perform the loading operation. Returns the deserialized object or null if the load fails. |
| getSeed() | SeedString<K> | O(1) | Returns the resource identifier for this loading operation. |
| getDataFolder() | Path | O(1) | Returns the root directory from which the resource will be loaded. |

## Integration Patterns

### Standard Usage
The intended pattern is for a managing system to instantiate a concrete Loader, execute the load method, and then process the result. The Loader instance is typically discarded immediately after use.

```java
// A hypothetical AssetManager using a concrete JsonObjectLoader
SeedString<MyResource> resourceSeed = SeedString.of("my_resource_key");
Path assetsRoot = Paths.get("gamedata/assets");

// Loader is created for a single, specific operation
Loader<MyResource, JsonObject> loader = new JsonObjectLoader(resourceSeed, assetsRoot);

// Operation is executed on a worker thread
JsonObject result = assetExecutor.submit(loader::load).get();

if (result != null) {
    // Process the successfully loaded object
}
```

### Anti-Patterns (Do NOT do this)
- **Main Thread Execution:** Never call the load method directly on the main game thread. This will block the thread, leading to application freezes.
- **Ignoring Nulls:** The load method is explicitly annotated as Nullable. Its contract allows for load failures (e.g., file not found, parsing error). Code that calls load **must** perform a null check on the result.
- **Instance Caching:** Do not cache and reuse Loader instances. They are lightweight, single-purpose objects. Caching them provides no benefit and can lead to bugs if the underlying file system state changes.

## Data Pipeline
The Loader acts as a critical step in the data pipeline that transforms a resource identifier into a usable in-memory object.

> Flow:
> Resource Request (SeedString) -> Asset Manager -> **Loader<K, T> (Instantiation)** -> Filesystem I/O -> Deserialization Logic -> Loaded Object (T) or null

