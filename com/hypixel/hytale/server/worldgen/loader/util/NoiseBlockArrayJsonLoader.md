---
description: Architectural reference for NoiseBlockArrayJsonLoader
---

# NoiseBlockArrayJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.util
**Type:** Transient

## Definition
```java
// Signature
public class NoiseBlockArrayJsonLoader extends JsonLoader<SeedStringResource, NoiseBlockArray> {
```

## Architecture & Concepts
The NoiseBlockArrayJsonLoader is a specialized deserializer within the server's world generation framework. Its primary function is to translate a specific JSON structure from a world generation configuration file into a runtime `NoiseBlockArray` object. This class acts as a critical bridge between static, human-readable world generation data (defined by designers in JSON) and the in-memory data structures consumed by the procedural generation algorithms.

This loader is part of a composite pattern, where a high-level configuration loader parses a large file and delegates smaller, specific JSON fragments to specialized loaders like this one. It further delegates the parsing of its own sub-components (like individual entries, noise properties, and value ranges) to even more granular loaders such as `EntryJsonLoader`, `NoisePropertyJsonLoader`, and `DoubleRangeJsonLoader`. This hierarchical approach promotes separation of concerns and makes the world generation configuration highly extensible.

The loader is responsible for interpreting several JSON formats for a block array:
1.  A single primitive string for a simple, one-entry array.
2.  A single JSON object for a one-entry array with complex properties.
3.  A JSON array of objects for a multi-entry array.

## Lifecycle & Ownership
-   **Creation:** An instance is created on-the-fly by a parent configuration loader during the world generation asset loading phase. It is passed the specific `JsonElement` it is responsible for, along with a `SeedString` for deterministic procedural generation and the path to the data folder for resolving any external references.
-   **Scope:** The object's lifetime is exceptionally short. It is designed to be single-use and exists only for the duration of the `load()` method call.
-   **Destruction:** Once the `load()` method returns the fully constructed `NoiseBlockArray` object, the NoiseBlockArrayJsonLoader instance has served its purpose and is immediately eligible for garbage collection. It holds no persistent state beyond the scope of the load operation.

## Internal State & Concurrency
-   **State:** The internal state of the loader (`seed`, `dataFolder`, `json`) is provided entirely via its constructor and is treated as immutable. The class reads this state to produce its output but does not modify it. It performs no internal caching.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated and used within a single, controlled thread, typically the main server thread during the world asset loading sequence. There are no internal locks or synchronization primitives, and concurrent access would lead to unpredictable behavior.

## API Surface
The public contract is minimal, focusing exclusively on the loading operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NoiseBlockArrayJsonLoader(seed, dataFolder, json) | constructor | O(1) | Creates a new loader instance for a specific JSON element. |
| load() | NoiseBlockArray | O(N) | Parses the JSON and constructs the NoiseBlockArray. N is the number of entries in the JSON array. Throws Error or IllegalArgumentException on malformed data. |

## Integration Patterns

### Standard Usage
This loader is not intended to be used directly. It is invoked by a higher-level system responsible for loading world generation profiles. The calling code is expected to handle exceptions to provide clear error messages about which configuration file is invalid.

```java
// A parent loader has parsed a larger file and has a JsonElement for a block array
JsonElement blockArrayJson = worldGenProfileJson.get("surfaceBlocks");
SeedStringResource currentSeed = ...;
Path dataPath = ...;

// Delegate parsing to the specialized loader
NoiseBlockArrayJsonLoader loader = new NoiseBlockArrayJsonLoader(currentSeed, dataPath, blockArrayJson);
NoiseBlockArray surfaceBlocks = loader.load();

// Use the resulting object in the world generator
biome.setSurfaceComposition(surfaceBlocks);
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not attempt to reuse a loader instance. It is intrinsically bound to the `JsonElement` provided in its constructor. For each new element to parse, create a new loader instance.
-   **External Exception Handling:** The `load` method can throw critical exceptions on invalid data. Failure to catch these exceptions at a high level will result in a server crash, masking the root cause, which is typically a typo in a JSON configuration file.
-   **Concurrent Loading:** Do not invoke the `load` method from multiple threads on the same instance. All world generation asset loading should be performed synchronously during server startup.

## Data Pipeline
This component sits in the middle of the world generation asset loading pipeline, transforming raw text data into a usable engine object.

> Flow:
> WorldGen JSON File -> GSON Parser -> Parent Config Loader -> **NoiseBlockArrayJsonLoader** -> In-Memory `NoiseBlockArray` Object -> Biome/Layer Generator Algorithm

