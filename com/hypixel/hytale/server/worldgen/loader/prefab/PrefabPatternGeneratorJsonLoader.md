---
description: Architectural reference for PrefabPatternGeneratorJsonLoader
---

# PrefabPatternGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.prefab
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPatternGeneratorJsonLoader extends JsonLoader<SeedStringResource, PrefabPatternGenerator> {
```

## Architecture & Concepts
The PrefabPatternGeneratorJsonLoader is a specialized factory class responsible for deserializing a JSON definition into a fully configured PrefabPatternGenerator object. It acts as a critical bridge between the declarative data assets that define world generation rules and the procedural, in-memory objects that execute those rules.

This loader is a key component of the server's world generation asset pipeline. Its primary architectural function is to orchestrate the loading of numerous, complex sub-components that constitute a prefab placement strategy. It achieves this through a **Composition of Loaders** pattern, delegating the parsing of specific JSON sections to other specialized loaders, such as PointGeneratorJsonLoader, HeightThresholdInterpreterJsonLoader, and NoiseMaskConditionJsonLoader.

By encapsulating the complex logic of parsing and object hydration, this class allows world designers to define intricate prefab placement behaviors in a simple, human-readable JSON format without needing to interact with the underlying Java implementation.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level asset management system during the server's world generation bootstrap phase. It is instantiated with a specific JSON element to parse, a file context for resolving dependencies, and a seed for deterministic procedural generation. It is not intended for manual instantiation during the game loop.

- **Scope:** The object's lifetime is ephemeral and confined to a single load operation. It exists only for the duration required to parse the JSON and construct the resulting PrefabPatternGenerator.

- **Destruction:** Once the `load` method returns, the PrefabPatternGeneratorJsonLoader instance has fulfilled its purpose and holds no further state. It becomes eligible for standard garbage collection. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The internal state of the loader consists of immutable references to the `seed`, `dataFolder`, `json`, and `context` provided during construction. It does not cache data or maintain mutable state between method calls. All loaded components are aggregated into a new object and returned, not stored internally.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated and used within a single, synchronous asset loading task. Concurrent calls to the `load` method on the same instance will result in undefined behavior. The asset loading pipeline is responsible for ensuring that each loader instance is confined to a single thread.

## API Surface
The public API is minimal, exposing only the primary `load` method and a static utility for a common sub-task.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | PrefabPatternGenerator | O(N) | The primary entry point. Parses the JSON and constructs the generator. N is the number of sub-components. Throws IllegalArgumentException on malformed or missing JSON keys. |
| loadRotations(JsonElement) | static PrefabRotation[] | O(K) | A static utility method to parse a JSON element into an array of prefab rotations. K is the number of rotation entries. |

## Integration Patterns

### Standard Usage
This loader is intended to be used by a parent system that manages the loading of world generation assets from disk. The parent system is responsible for reading the file, parsing it into a JSON tree, and then invoking the loader.

```java
// Hypothetical usage within a WorldGen asset manager
FileLoadingContext context = ...;
SeedStringResource seed = ...;
Path dataFolder = ...;
JsonElement prefabPatternJson = loadJsonFromFile("my_dungeon_pattern.json");

// Instantiate the loader for a single, specific task
PrefabPatternGeneratorJsonLoader loader = new PrefabPatternGeneratorJsonLoader(seed, dataFolder, prefabPatternJson, context);

// The resulting object is now ready for use by the world generator
PrefabPatternGenerator generator = loader.load();
worldGenerator.registerPattern(generator);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not hold a reference to a loader instance to use it multiple times. Each load operation for a distinct JSON file or element requires a new instance.

- **Game Loop Instantiation:** Never instantiate this class during the main server game loop. It is part of the initial, blocking asset loading pipeline. Instantiating it on-the-fly can cause performance stalls due to file I/O and complex object graph creation.

- **Invalid Context:** Providing an incomplete or incorrect FileLoadingContext can lead to runtime errors. For example, if the context is missing a referenced PrefabCategory, the loader will log a warning and default to a safe value, which may not be the intended behavior.

## Data Pipeline
The PrefabPatternGeneratorJsonLoader is a transformation step in the data flow from static configuration files to live world generation logic.

> Flow:
> JSON File on Disk -> GSON Parser -> JsonElement -> **PrefabPatternGeneratorJsonLoader** -> PrefabPatternGenerator Object -> World Generation Service

