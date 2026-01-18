---
description: Architectural reference for CellNoiseJsonLoader
---

# CellNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class CellNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseFunction> {
```

## Architecture & Concepts

The CellNoiseJsonLoader is a specialized factory component within the procedural generation framework. Its primary responsibility is to deserialize a declarative JSON configuration into a fully-realized, executable NoiseFunction object, specifically an instance of CellNoise.

This loader operates as a key bridge between the static data assets that define world generation and the live, in-memory objects that perform the generation. It embodies the **Composite Loader Pattern**; rather than parsing the entire JSON structure monolithically, it acts as an orchestrator. It delegates the responsibility of parsing complex sub-components—such as distance functions, point evaluators, and noise lookups—to other, more specialized JsonLoader implementations (e.g., CellDistanceFunctionJsonLoader, PointEvaluatorJsonLoader). This design promotes high cohesion, low coupling, and simplifies the maintenance of individual noise components.

A critical architectural detail is that the loader does not produce a generic CellNoise object. Instead, it instantiates a private inner class, LoadedCellNoise. This specialized subclass maintains a reference to a SeedResource, which it uses to acquire thread-local result buffers. This is a significant performance and concurrency optimization, ensuring that the generated NoiseFunction can be safely and efficiently executed by multiple world-generation worker threads without incurring lock contention or requiring new buffer allocations for every operation.

## Lifecycle & Ownership

-   **Creation:** An instance of CellNoiseJsonLoader is created on-demand by a higher-level orchestrator, typically a master noise loading service that dispatches to the correct loader based on a "type" field within a given JSON asset. It is not intended for direct instantiation by end-users.
-   **Scope:** The object's lifetime is exceptionally brief. It is designed to be single-use, existing only for the duration of the call to its `load` method. It holds no state that persists beyond this operation.
-   **Destruction:** The loader instance becomes eligible for garbage collection immediately after the `load` method returns its NoiseFunction product. All constructor-provided dependencies (JsonElement, SeedString) are owned by the calling system, not the loader.

## Internal State & Concurrency

-   **State:** The internal state of the loader (seed, data folder, and the source JSON) is provided at construction and is treated as immutable. The loader itself is stateless in the sense that it does not modify its own configuration during the loading process.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, used for a single `load` operation, and then discarded, all within the confines of a single thread.

    **WARNING:** While the loader itself is confined to a single thread, the `NoiseFunction` object it produces (*LoadedCellNoise*) is explicitly designed to be thread-safe. It achieves this by retrieving thread-local buffers from its associated SeedResource, allowing for concurrent evaluation by multiple generator threads. Confusing the thread safety of the loader with its product can lead to severe concurrency bugs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseFunction | O(N) | Parses the configured JSON and constructs a complete NoiseFunction. N is the number of nodes in the JSON sub-tree. Throws JsonSyntaxException on malformed input. |

## Integration Patterns

### Standard Usage

This loader is an internal component and should not be invoked directly. It is used by a parent loading system that interprets procedural asset files.

```java
// Hypothetical usage by a master loader
JsonElement noiseConfig = readNoiseFile("my_cellular_noise.json");
SeedString seed = ...;
Path dataFolder = ...;

// The master loader would identify the type and delegate
if (isCellNoise(noiseConfig)) {
    // The loader is created, used, and immediately discarded
    NoiseFunction cellNoiseFunc = new CellNoiseJsonLoader<>(seed, dataFolder, noiseConfig).load();
    
    // The resulting function is now ready for use in world generation
    worldGenerator.registerNoiseFunction("terrain_base", cellNoiseFunc);
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Caching:** Do not attempt to cache and reuse a CellNoiseJsonLoader instance. Each instance is bound to a specific JsonElement and is not designed for repeated `load` calls or reconfiguration.
-   **Direct Instantiation:** Application-level code, such as game logic or mod scripts, must not instantiate this class directly. All noise functions should be retrieved from a central procedural generation registry or service, which abstracts away the loading details.
-   **State Mutation:** Do not attempt to modify the loader's state via reflection after construction. The loading process assumes its initial configuration is immutable.

## Data Pipeline

The flow of data from a static asset file to a usable runtime object follows a clear path of deserialization and composition.

> Flow:
> WorldGen Asset (*.json file*) -> JSON Parser (Gson) -> `JsonElement` -> Master Noise Registry -> **CellNoiseJsonLoader** -> `LoadedCellNoise` (as `NoiseFunction`) -> World Generation Engine

