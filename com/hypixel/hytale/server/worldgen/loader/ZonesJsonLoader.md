---
description: Architectural reference for ZonesJsonLoader
---

# ZonesJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader
**Type:** Transient

## Definition
```java
// Signature
public class ZonesJsonLoader extends Loader<SeedStringResource, Zone[]> {
```

## Architecture & Concepts
The ZonesJsonLoader is a high-level orchestrator within the server's world generation framework. It is not responsible for the low-level parsing of individual zone files; instead, its primary function is to manage the *batch loading* of all zone definitions required for world generation.

It operates as a master loader, consuming a `FileLoadingContext` which acts as a manifest of all known zone directories. For each entry in this manifest, it delegates the actual file parsing to a more specialized `ZoneJsonLoader` instance. This separation of concerns follows the Composite pattern: ZonesJsonLoader manages the collection, while ZoneJsonLoader manages the individual item.

This class forms a critical link between the file system representation of world generation content and the in-memory `Zone` objects used by the procedural generation engine at runtime.

### Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level bootstrap process, such as a world or dimension loader. The caller is responsible for supplying a fully populated `FileLoadingContext`, a root `dataFolder` path, and a procedural `seed`.
- **Scope:** Short-lived and task-scoped. An instance of ZonesJsonLoader exists only for the duration of a single `load` operation. It is created, its `load` method is invoked, and it is then immediately eligible for garbage collection.
- **Destruction:** The object is destroyed by the garbage collector once the calling scope completes and the returned `Zone[]` array is the only artifact that persists.

## Internal State & Concurrency
- **State:** The class is stateful, but its state is provided at construction and treated as immutable for the instance's lifetime. It holds references to the `loadingContext`, `seed`, and `dataFolder`. It does not cache any data between calls or across instances. The `load` method's internal state, such as the results array, is confined to the method's stack frame.
- **Thread Safety:** This class is **not thread-safe** and is designed for synchronous execution. The `load` method performs blocking file I/O operations and must not be called on a performance-critical thread like the main server tick loop. If asynchronous loading is required, the entire ZonesJsonLoader instance should be managed by a dedicated worker thread or task scheduler.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | Zone[] | O(N * M) | Orchestrates the loading of all zones. N is the number of zones in the context, and M is the complexity of parsing a single zone file. This is a blocking I/O operation. Throws an `Error` if any zone fails to load, halting the entire process. |

## Integration Patterns

### Standard Usage
The loader is intended to be used as a single-shot utility. The caller prepares the context, instantiates the loader, executes the load, and then uses the resulting array.

```java
// 1. Obtain a pre-populated loading context from a discovery service.
FileLoadingContext loadingContext = worldAssetScanner.discoverWorldGenFiles();

// 2. Obtain the world seed and data folder path.
SeedString<SeedStringResource> seed = world.getSeed();
Path dataFolder = server.getDataFolderPath();

// 3. Instantiate the loader and execute the operation.
ZonesJsonLoader loader = new ZonesJsonLoader(seed, dataFolder, loadingContext);
Zone[] allAvailableZones = loader.load();

// 4. Pass the resulting array to the world generator.
worldGenerator.initialize(allAvailableZones);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of ZonesJsonLoader to call `load` multiple times. The external `FileLoadingContext` could change, leading to stale or inconsistent results. Always create a new loader for a fresh loading operation.
- **Empty Context:** Passing a default or empty `FileLoadingContext` to the constructor is valid but dangerous. The `load` method will successfully return an empty `Zone[]` array, which will likely cause `NullPointerException` or `ArrayIndexOutOfBoundsException` in downstream world generation systems. The context must be validated before creating the loader.
- **Error Swallowing:** The `load` method throws a critical `Error`, not a checked `Exception`. This is by design, as a failure to load a zone is considered a fatal configuration error. Do not wrap the call in a generic `try/catch (Throwable t)` block that silences the issue. A failure here indicates a corrupt or incomplete server installation that must be fixed.

## Data Pipeline
The flow of data is from a high-level manifest down to raw files, which are then parsed and aggregated back into a final in-memory collection.

> Flow:
> `FileLoadingContext` (In-memory manifest of zone paths) -> **ZonesJsonLoader** (Receives context) -> Iterates and dispatches to `ZoneJsonLoader` -> Reads `Zone.json` from disk -> `ZoneJsonLoader` parses JSON into a `Zone` object -> **ZonesJsonLoader** (Aggregates `Zone` objects into a `Zone[]` array) -> World Generation Engine

