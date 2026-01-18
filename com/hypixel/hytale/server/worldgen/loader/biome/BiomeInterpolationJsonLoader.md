---
description: Architectural reference for BiomeInterpolationJsonLoader
---

# BiomeInterpolationJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.biome
**Type:** Transient

## Definition
```java
// Signature
public class BiomeInterpolationJsonLoader extends JsonLoader<SeedStringResource, BiomeInterpolation> {
```

## Architecture & Concepts
The BiomeInterpolationJsonLoader is a specialized parser responsible for translating a declarative JSON configuration into a runtime-optimized BiomeInterpolation object. It operates as a key component within the server's world generation pipeline, specifically during the loading and interpretation of zone files.

This class embodies the **Builder** pattern. It takes a raw, structured data format (JSON) and constructs a complex, stateful Java object used by the procedural generation engine. Its primary function is to define the "smoothing radius" for biome transitions, allowing for fine-grained control over how different biomes blend together. A larger radius results in a more gradual, smoother transition, while a smaller radius creates sharper, more defined borders.

It depends on a ZoneFileContext to resolve biome identifiers and other contextual data, making it a context-aware loader. For complex sub-tasks, such as parsing biome masks, it delegates to other specialized loaders like BiomeMaskJsonLoader, promoting a clear separation of concerns.

## Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a higher-level world generation loader, typically the one responsible for parsing an entire zone configuration file. It is passed the specific JSON element corresponding to the biome interpolation rules.
- **Scope:** The object's lifetime is extremely short and task-specific. It exists only for the duration of the `load` method call.
- **Destruction:** Once the `load` method returns the constructed BiomeInterpolation object, the loader instance has served its purpose and is eligible for standard garbage collection. It holds no persistent state and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The loader's state (the input JSON, data folder path, and ZoneFileContext) is provided at construction and is treated as immutable. The `load` method is a pure transformation of this initial state into a BiomeInterpolation object.
- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. It is expected to be used exclusively within the main world generation loading thread. Concurrent invocation would be an architectural violation, as the worldgen asset loading phase is not designed for parallel execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | BiomeInterpolation | O(N) | Parses the JSON configuration and constructs a BiomeInterpolation object. N is the number of biome-specific rules. Throws Error on malformed data, missing properties, or duplicate biome entries. |

## Integration Patterns

### Standard Usage
This loader is not a managed service and should be instantiated directly by a parent loader that has access to the zone file's JSON structure and the necessary world generation context.

```java
// Example from a hypothetical ZoneFileParser
JsonElement interpolationJson = zoneConfig.get("BiomeInterpolation");
ZoneFileContext context = this.getZoneContext();
SeedString<SeedStringResource> seed = this.getSeed();

BiomeInterpolationJsonLoader loader = new BiomeInterpolationJsonLoader(seed, dataFolder, interpolationJson, context);
BiomeInterpolation interpolationRules = loader.load();

// The 'interpolationRules' object is now passed to the world generator
worldGenerator.setBiomeInterpolation(interpolationRules);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not attempt to cache and re-use a loader instance. Each instance is bound to the specific JSON element provided at construction. To load another configuration, create a new instance.
- **Context-less Operation:** Providing a null or incomplete ZoneFileContext will result in runtime errors. The loader relies on this context to correctly interpret biome masks and other identifiers.
- **Error Swallowing:** The loader throws unrecoverable Error exceptions for configuration issues. These are considered critical failures in the world generation pipeline and must not be caught and ignored, as it would result in a corrupt or unpredictable game world.

## Data Pipeline
The loader acts as a transformation step, converting human-readable configuration into an efficient, integer-mapped data structure for the procedural engine.

> Flow:
> Zone File (JSON) -> Parent Zone Loader -> **BiomeInterpolationJsonLoader** -> BiomeInterpolation (In-Memory Object) -> Biome Placement Algorithm

