---
description: Architectural reference for NoisePropertyJsonLoader
---

# NoisePropertyJsonLoader<K>

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class NoisePropertyJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseProperty> {
```

## Architecture & Concepts

The NoisePropertyJsonLoader is a specialized factory responsible for deserializing a JSON configuration into a fully realized, executable `NoiseProperty` object graph. It serves as the primary bridge between declarative world generation design (authored in JSON files) and the imperative code of the procedural generation library.

Architecturally, this loader implements a recursive descent parser. It traverses a `JsonElement` tree and, for each JSON object, constructs a corresponding `NoiseProperty` instance. This process is inherently compositional; complex noise functions defined in JSON as nested objects are materialized into a tree of `NoiseProperty` objects that wrap and modify each other. This allows for the creation of sophisticated, multi-layered noise through simple data configuration.

The loader's logic is bifurcated:
1.  **Core Object Creation:** It first identifies the fundamental noise type. If a `Type` field is specified, it uses a factory pattern to create a composite property (e.g., `MaxNoiseProperty`, `SumNoiseProperty`). If no `Type` is specified, it defaults to creating a base generator (`SingleNoiseProperty` or `FractalNoiseProperty`).
2.  **Decorator Application:** After the core object is instantiated, the loader checks for optional JSON keys (e.g., `Scale`, `Normalize`, `Offset`). If present, it wraps the newly created `NoiseProperty` in additional decorator objects (`ScaleNoiseProperty`, `NormalizeNoiseProperty`, etc.), layering on functionality.

This two-phase approach provides a flexible and powerful system for defining procedural noise without modifying engine code.

### Lifecycle & Ownership
-   **Creation:** An instance is created on-demand whenever a segment of the world generation configuration needs to be parsed into a `NoiseProperty`. This is a short-lived, single-purpose object. Parent loaders or the primary procedural bootstrap process are responsible for its instantiation.
-   **Scope:** The loader's lifetime is strictly confined to the execution of its `load` method. Once it returns the constructed `NoiseProperty` object, the loader instance itself has served its purpose and is eligible for garbage collection.
-   **Destruction:** Handled automatically by the Java Virtual Machine's garbage collector. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** The loader is stateful, containing references to the `seed`, `dataFolder`, and the specific `JsonElement` it is currently processing. This state provides the necessary context for the deserialization task. The state is considered volatile and is not intended to persist beyond the `load` call.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be used in a single-threaded context during the asset loading and world generation setup phase. The internal `SeedString` object may be modified during the `load` operation to generate deterministic seeds for child properties. Accessing a single instance from multiple threads will result in race conditions and non-deterministic output.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NoisePropertyJsonLoader(seed, dataFolder, json) | Constructor | O(1) | Creates a new loader for a specific JSON element and generation seed. |
| load() | NoiseProperty | O(N) | Parses the internal JSON element and constructs the complete `NoiseProperty` object graph. N is the number of nodes in the JSON tree. Throws `IllegalStateException` if the JSON is malformed or missing required keys. |

## Integration Patterns

### Standard Usage

This loader is not intended for direct use by game logic developers. It is invoked by higher-level configuration management systems within the procedural generation engine. The standard pattern involves creating a new loader for each `NoiseProperty` definition encountered.

```java
// Pseudo-code demonstrating engine-level usage
JsonElement noiseConfig = loadJsonFromFile("worldgen/biomes/forest_heightmap.json");
SeedString<WorldSeed> seed = world.getSeed().append("ForestHeight");
Path dataFolder = world.getDataFolderPath();

// Create a loader for this specific configuration and load it
NoisePropertyJsonLoader<WorldSeed> loader = new NoisePropertyJsonLoader<>(seed, dataFolder, noiseConfig);
NoiseProperty heightMap = loader.load();

// The resulting heightMap object is now ready for use by the world generator
worldGenerator.setHeightMap(heightMap);
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not cache and re-use a `NoisePropertyJsonLoader` instance. Each instance is tied to a specific `JsonElement` and its internal state is not designed for reset. Always create a new loader for each distinct JSON object to be parsed.
-   **Concurrent Loading:** Do not share a single loader instance across multiple threads. To parse multiple noise configurations in parallel, create a separate `NoisePropertyJsonLoader` instance for each thread.
-   **State Manipulation:** Do not attempt to modify the internal state of the loader after construction. The object should be treated as immutable from the caller's perspective once created.

## Data Pipeline

The `NoisePropertyJsonLoader` is a critical transformation step in the world generation data pipeline. It converts static data into executable logic.

> Flow:
> JSON Configuration File -> GSON Parser -> `JsonElement` -> **NoisePropertyJsonLoader** -> `NoiseProperty` Object Graph -> World Generator Engine

