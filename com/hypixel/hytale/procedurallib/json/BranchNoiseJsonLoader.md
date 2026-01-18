---
description: Architectural reference for BranchNoiseJsonLoader
---

# BranchNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class BranchNoiseJsonLoader<T extends SeedResource> extends AbstractCellJitterJsonLoader<T, BranchNoise> {
```

## Architecture & Concepts

The BranchNoiseJsonLoader is a factory class that serves a single, critical purpose: to deserialize a JSON configuration object into a fully-realized, executable BranchNoise instance. It acts as a bridge between the data-driven design of the procedural generation system and the concrete Java objects that perform the noise calculations.

This loader is a specialized component within a larger framework of JSON deserializers, as evidenced by its inheritance from AbstractCellJitterJsonLoader. It encapsulates the complex and error-prone logic of parsing numerous parameters related to cellular noise, applying sensible defaults for missing values, and assembling the final, complex BranchNoise object.

The core design principle is the translation of declarative data (JSON) into imperative code (a configured BranchNoise object). By using a generic type T that extends SeedResource, the loader ensures that the resulting noise algorithm is correctly bound to a specific world generation context, which includes the world seed and, critically, shared, thread-local buffers for performance.

## Lifecycle & Ownership

-   **Creation:** An instance of BranchNoiseJsonLoader is created on-demand by a higher-level configuration manager or world generator. This typically occurs during an initial loading phase when procedural generation assets are being parsed from files.
-   **Scope:** The loader's scope is extremely brief and transactional. It is designed to be instantiated, have its `load` method called exactly once, and then be immediately discarded. It holds no state that is relevant beyond this single operation.
-   **Destruction:** The loader object becomes eligible for garbage collection as soon as the `load` method returns and its reference goes out of scope. The caller who invoked `load` assumes ownership of the returned BranchNoise object, which will have a much longer lifecycle, often persisting for the duration of the world generation process.

## Internal State & Concurrency

-   **State:** The internal state of the loader consists of the seed, data folder path, and the input JsonElement. This state is provided at construction and is treated as immutable for the lifetime of the object. The loader is stateless with respect to the loading process itself; it performs no caching between method calls.

-   **Thread Safety:** The BranchNoiseJsonLoader class is **not thread-safe** and must not be shared across threads. It is designed for single-threaded access within a broader asset loading pipeline.

    **WARNING:** While the loader itself is not thread-safe, the `LoadedBranchNoise` object it produces **is designed for concurrent execution**. This is a critical architectural distinction. Thread safety in the resulting object is achieved by delegating buffer management to the provided SeedResource, which supplies thread-local buffers via the `localBuffer2d` method. This prevents contention when multiple world generation threads evaluate the noise function simultaneously.

## API Surface

The public contract is minimal, focusing exclusively on the creation and execution of the loading process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BranchNoiseJsonLoader(seed, dataFolder, json) | Constructor | O(1) | Instantiates the loader with the required context for a single deserialization operation. |
| load() | BranchNoise | O(N) | The primary entry point. Parses the internal JSON, constructs, and returns a new LoadedBranchNoise instance. Complexity is proportional to the number of keys in the JSON definition. |

## Integration Patterns

### Standard Usage

The loader is intended to be used as part of a larger configuration loading system. The caller is responsible for parsing the root JSON file and passing the relevant JsonElement to the loader.

```java
// Assume a manager class that orchestrates loading
JsonElement branchNoiseJson = rootConfig.get("riverBranchNoiseDefinition");
SeedResource resource = worldContext.getSeedResource();
Path dataPath = worldContext.getDataFolderPath();

// Create, use, and discard the loader in one logical block
BranchNoiseJsonLoader<SeedResource> loader = new BranchNoiseJsonLoader<>(resource.getSeedString(), dataPath, branchNoiseJson);
BranchNoise riverNoise = loader.load();

// The resulting noise object is now ready for use in the world generator
worldGenerator.registerNoiseFunction("rivers", riverNoise);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Reuse:** Do not retain and reuse a BranchNoiseJsonLoader instance to load multiple, different JSON objects. Its internal state is tied to the JsonElement provided at construction. A new loader must be created for each distinct definition.
-   **Concurrent Loading:** Do not share a single loader instance across multiple threads to parse configurations in parallel. This will result in race conditions and unpredictable behavior.
-   **State Modification:** Do not attempt to modify the loader's internal state after construction. It is designed as a "fire-and-forget" transactional object.

## Data Pipeline

The BranchNoiseJsonLoader is a key transformation step in the procedural asset pipeline, converting raw data into an executable algorithm.

> Flow:
> JSON File on Disk -> GSON Parser -> JsonElement -> **BranchNoiseJsonLoader** -> LoadedBranchNoise Instance -> World Generation System

