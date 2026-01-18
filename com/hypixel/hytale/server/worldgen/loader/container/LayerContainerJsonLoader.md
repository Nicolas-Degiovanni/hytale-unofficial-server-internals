---
description: Architectural reference for LayerContainerJsonLoader
---

# LayerContainerJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.container
**Type:** Transient

## Definition
```java
// Signature
public class LayerContainerJsonLoader extends JsonLoader<SeedStringResource, LayerContainer> {
```

## Architecture & Concepts
The LayerContainerJsonLoader is a specialized deserializer that acts as a critical bridge between the declarative world generation data assets (defined in JSON) and the procedural generation engine. Its sole responsibility is to parse a complex JSON structure and transform it into a fully hydrated, in-memory LayerContainer object. This object defines the vertical composition of terrain for a given region, specifying layers of materials like stone, dirt, and grass.

This loader employs a compositional design pattern, utilizing a hierarchy of nested static loader classes (StaticLayerJsonLoader, DynamicLayerJsonLoader, etc.). Each nested loader is responsible for a specific, isolated part of the input JSON structure. This approach contains complexity, making the parsing logic more modular and maintainable. It delegates the parsing of common procedural primitives, such as noise functions and value ranges, to specialized loaders from the procedurallib library.

During parsing, it resolves human-readable asset identifiers (e.g., "hytale:stone") into optimized integer IDs by querying global asset registries like BlockType.getAssetMap(). This translation is essential for the performance of the core world generation algorithms, which operate on numerical data.

### Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a higher-level asset loading system, typically when a biome or world zone definition is being processed. The caller must provide the raw JsonElement to be parsed, along with a world seed and data path context.
- **Scope:** The object is ephemeral and has a very short-lived scope. Its existence is tied directly to the execution of a single `load` operation.
- **Destruction:** It is eligible for garbage collection immediately after the `load` method returns its LayerContainer result. State is not persisted within the loader instance.

## Internal State & Concurrency
- **State:** The internal state (seed, dataFolder, json) is provided at construction and is treated as immutable for the object's lifetime. The loader is stateful but designed for a single, non-reentrant operation. It does not cache data beyond the scope of the `load` method call.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution during the asset loading phase of world generation. Concurrent calls to `load` on the same instance will result in undefined behavior.

## API Surface
The public API is intentionally minimal, exposing only the primary factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | LayerContainer | O(N) | Parses the configured JSON data and constructs a new LayerContainer. N is the number of nodes in the JSON tree. Throws Error or IllegalArgumentException on malformed data or missing asset references. |

## Integration Patterns

### Standard Usage
This loader is intended to be used by other parts of the world generation system that need to parse a layer container definition from a larger JSON file.

```java
// Assume 'biomeJson' is a JsonElement representing a biome asset
// and 'context' provides the necessary seed and paths.

JsonElement layerData = biomeJson.get("layers");
SeedStringResource seed = context.getSeed().append("biomeLayers");
Path dataPath = context.getDataPath();

// Instantiate, load, and use the result. The loader is then discarded.
LayerContainerJsonLoader loader = new LayerContainerJsonLoader(seed, dataPath, layerData);
LayerContainer container = loader.load();

worldGenerator.applyLayers(container);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not call the `load` method more than once on a single instance. Each parsing operation requires a new loader instance.
- **Multi-threaded Access:** Do not share an instance of this loader across multiple threads. The internal parsing state is not protected by synchronization mechanisms.
- **Creation Without Context:** Instantiating the loader with a null or incomplete JsonElement will result in immediate exceptions upon calling `load`.

## Data Pipeline
The LayerContainerJsonLoader functions as a transformation step within the broader world generation asset pipeline. It consumes raw data and produces a structured, engine-ready object.

> Flow:
> World Generation Asset (JSON File) -> Asset Loading Service -> JsonElement -> **LayerContainerJsonLoader** -> LayerContainer (In-Memory Object) -> Procedural Generation Engine

