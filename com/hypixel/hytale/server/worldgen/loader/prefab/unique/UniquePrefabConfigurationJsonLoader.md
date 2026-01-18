---
description: Architectural reference for UniquePrefabConfigurationJsonLoader
---

# UniquePrefabConfigurationJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.prefab.unique
**Type:** Transient

## Definition
```java
// Signature
public class UniquePrefabConfigurationJsonLoader extends JsonLoader<SeedStringResource, UniquePrefabConfiguration> {
```

## Architecture & Concepts
The UniquePrefabConfigurationJsonLoader is a specialized deserializer responsible for transforming a JSON configuration block into a strongly-typed UniquePrefabConfiguration object. It operates within the server-side world generation pipeline, specifically during the parsing of zone configuration files.

This class acts as a compositional builder. Rather than containing all parsing logic itself, it delegates the deserialization of complex sub-objects—such as biome masks, noise masks, and vector data—to other, more focused JsonLoader implementations (e.g., BiomeMaskJsonLoader, NoiseMaskConditionJsonLoader). This adheres to the Single Responsibility Principle, keeping the loader focused solely on the structure of a unique prefab configuration.

The loader is context-aware, requiring a ZoneFileContext during construction. This context is critical for resolving references within the JSON, such as biome names, which are specific to the zone being generated. This design ensures that prefab configurations can be validated against the environment they are intended for at load time.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level component in the world generation system, likely a master zone file parser. When this parent parser encounters a JSON object defining a unique prefab, it instantiates this loader, providing it with the relevant JSON snippet and the current world generation context.
- **Scope:** The object's lifetime is extremely short and bound to a single operation. It is created, its load method is called once, and it is then immediately eligible for garbage collection. It is a classic transient, single-use object.
- **Destruction:** The instance is destroyed by the Java garbage collector once the calling scope completes and the reference is dropped. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The internal state consists of the world seed, data folder path, the target JsonElement, and the ZoneFileContext. This state is provided via the constructor and is treated as immutable for the object's lifetime. The class does not cache results or modify its state after construction.
- **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. Each instance should be confined to the single thread responsible for parsing its specific JSON object. The world generation system may be multi-threaded, but it must ensure that each loader instance is used in a thread-local manner.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | UniquePrefabConfiguration | O(N) | Deserializes the provided JSON into a configuration object. N is the number of keys in the JSON. Throws IllegalArgumentException if the required *Anchor* key is missing. Throws Error if an *Environment* ID cannot be resolved. |

## Integration Patterns

### Standard Usage
The loader is instantiated with all necessary context and used immediately to produce a configuration object. This object is then passed to other world generation systems responsible for prefab placement.

```java
// Assume 'zoneParser' is processing a file and has the jsonObject for a prefab
ZoneFileContext context = zoneParser.getContext();
SeedString<SeedStringResource> seed = zoneParser.getSeed();
Path dataFolder = zoneParser.getDataFolder();

// Create the loader for a specific JSON block
UniquePrefabConfigurationJsonLoader loader = new UniquePrefabConfigurationJsonLoader(seed, dataFolder, jsonObject, context);

// Immediately load the configuration and pass it on
UniquePrefabConfiguration config = loader.load();
prefabPlacementService.register(config);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of this loader to parse multiple, different JSON objects. It is designed to be single-use. Create a new loader for each configuration block.
- **Incomplete Context:** Instantiating the loader with a null or improperly configured ZoneFileContext will lead to runtime exceptions if the JSON data contains context-dependent values like biome names.
- **Ignoring Exceptions:** The load method performs validation and will throw exceptions for malformed or incomplete data (e.g., a missing *Anchor*). These exceptions must be caught and handled gracefully, typically by logging an error and preventing the invalid prefab from being added to the world.

## Data Pipeline
This class is a critical transformation step in the data flow of world generation, converting raw, unstructured data into a validated, in-memory representation.

> Flow:
> Zone Configuration File (JSON) -> Server Zone Parser -> **UniquePrefabConfigurationJsonLoader** -> UniquePrefabConfiguration (Java Object) -> Prefab Placement System

