---
description: Architectural reference for WeightedPrefabMapJsonLoader
---

# WeightedPrefabMapJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.prefab
**Type:** Transient

## Definition
```java
// Signature
public class WeightedPrefabMapJsonLoader extends JsonLoader<SeedStringResource, IWeightedMap<WorldGenPrefabSupplier>> {
```

## Architecture & Concepts
The WeightedPrefabMapJsonLoader is a specialized parser within the server's world generation framework. It serves as a critical bridge between human-readable JSON configuration files and the engine's internal procedural generation data structures. Its primary responsibility is to interpret a specific JSON schema that defines a collection of "prefabs" (pre-designed environmental assets) and their corresponding spawn probabilities or weights.

This class is not a standalone service but rather a transient, single-purpose tool. It is invoked by higher-level world generation loaders that process broader zone or biome configuration files. By extending the generic JsonLoader, it inherits foundational JSON parsing capabilities and context, but specializes them for the task of building an IWeightedMap. This map is a probabilistic collection that the world generator can efficiently sample to select which prefab to place in the world, respecting the weights defined by designers.

Architecturally, it decouples the world generation algorithms from the raw data format. The generator operates on the abstract IWeightedMap interface, unaware of the JSON source, while this loader encapsulates the specific logic for parsing prefab and weight arrays.

### Lifecycle & Ownership
- **Creation:** An instance is created dynamically by a parent loading process when it encounters a JSON object that needs to be interpreted as a weighted prefab map. The constructor is supplied with the necessary context, including the specific JsonElement to parse, the data folder for resolving resources, and a SeedStringResource which provides access to the master WorldGenPrefabLoader.
- **Scope:** The object's lifetime is extremely brief. It exists only for the duration of the `load` method call. It is a stateless transformer.
- **Destruction:** Once the `load` method returns the constructed IWeightedMap, the WeightedPrefabMapJsonLoader instance has fulfilled its purpose and is immediately eligible for garbage collection. It holds no resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state of this class is immutable after construction. It stores references to the JSON keys (`prefabsKey`, `weightsKey`) and the context provided by its parent JsonLoader. The `load` method itself is state-transformative but does not mutate the loader instance; it produces a new IWeightedMap on each invocation.
- **Thread Safety:** This class is **not thread-safe** and is intended for single-threaded use. It is designed to be instantiated and used within the scope of a single loading task. Concurrent calls to the `load` method on the same instance would result in undefined behavior, although the transient nature of the class makes such a scenario highly unlikely in practice.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | IWeightedMap | O(N * M) | Parses the JSON, resolves prefab names via the WorldGenPrefabLoader, and builds the weighted map. Throws IllegalArgumentException if JSON is malformed, keys are missing, or resolved prefabs are empty. N is the number of prefab entries; M is the average number of suppliers per entry. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by high-level game logic. It is a component within the world generation data loading pipeline. A parent loader is responsible for its instantiation and invocation.

```java
// Context: Inside a higher-level worldgen configuration parser
JsonElement prefabMapJson = zoneConfig.get("structureSpawns");
SeedStringResource resourceContext = worldGenContext.getSeedStringResource();
Path dataFolder = worldGenContext.getDataFolderPath();

// Instantiate the loader for a specific JSON block
WeightedPrefabMapJsonLoader loader = new WeightedPrefabMapJsonLoader(
    resourceContext.getSeed(),
    dataFolder,
    prefabMapJson,
    "prefabs",
    "weights"
);

// Execute the load operation to get the final data structure
IWeightedMap<WorldGenPrefabSupplier> structureMap = loader.load();

// The map is now ready for use by the procedural generator
proceduralGenerator.setStructureMap(structureMap);
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not cache and reuse an instance of WeightedPrefabMapJsonLoader. It is a lightweight, single-use object. Create a new instance for each distinct JSON block you need to parse.
- **Premature Invocation:** Do not invoke `load` before the global WorldGenPrefabLoader (accessed via SeedStringResource) has been fully populated. Doing so will cause the loader to fail when it cannot resolve the prefab string identifiers.

## Data Pipeline
The loader is a key stage in the transformation of configuration data into a usable runtime object for the world generator.

> Flow:
> Zone JSON File -> Gson Parser -> `JsonElement` -> **`WeightedPrefabMapJsonLoader`** -> `WorldGenPrefabLoader` (for name resolution) -> `IWeightedMap<WorldGenPrefabSupplier>` -> Procedural World Generator

