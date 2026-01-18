---
description: Architectural reference for CustomBiomeJsonLoader
---

# CustomBiomeJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.biome
**Type:** Transient

## Definition
```java
// Signature
public class CustomBiomeJsonLoader extends BiomeJsonLoader {
```

## Architecture & Concepts
The CustomBiomeJsonLoader is a specialized factory component within the server's world generation pipeline. Its primary responsibility is to parse a JSON definition for a "custom biome" and transform it into a fully realized, in-memory CustomBiome object.

This class extends the generic BiomeJsonLoader, inheriting the foundational logic for parsing common biome properties like cover layers, prefabs, and environmental settings. Its key specialization lies in its ability to process and load the **CustomBiomeGenerator** configuration. This nested structure defines how a larger biome is composed from a set of smaller, predefined "tile" biomes, enabling complex and varied terrain without defining every detail from scratch.

Architecturally, this loader acts as a stateful builder during its short lifespan. It aggregates configuration data from a JsonElement and delegates the loading of nested complex objects, such as the CustomBiomeGenerator, to other specialized loaders. This compositional approach keeps the parsing logic modular and maintainable.

## Lifecycle & Ownership
- **Creation:** An instance of CustomBiomeJsonLoader is created by a higher-level world generation service (e.g., a BiomeRegistry loader) whenever it encounters a biome asset file that is designated as a custom biome. It is not a singleton; a new loader is instantiated for each biome definition to be processed.

- **Scope:** The loader is extremely short-lived. Its existence is scoped entirely to the duration of the `load` method call. Once the method returns the constructed CustomBiome object, the loader instance has fulfilled its purpose and holds no further relevant state.

- **Destruction:** The object is eligible for garbage collection immediately after the `load` method completes and its reference is dropped by the calling system. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state of the loader consists of final fields passed during construction, including the world seed, the source JSON, the biome context, and the array of available tile biomes. This state is effectively immutable after instantiation, making the loader's operation highly predictable.

- **Thread Safety:** This class is **not thread-safe** and is not intended for concurrent use. It is designed to be instantiated and used by a single thread as part of a sequential world generation bootstrap process. While its immutable state prevents internal data corruption, its reliance on file system context (`dataFolder`) and its single-shot design make concurrent `load` calls on a shared instance an unsupported and dangerous pattern.

## API Surface
The public contract is minimal, exposing only the constructor and the primary `load` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CustomBiomeJsonLoader(...) | Constructor | O(1) | Instantiates the loader. Requires all dependencies for biome creation, including the seed, JSON data, and the set of available tile biomes. |
| load() | CustomBiome | O(N) | The primary factory method. Orchestrates the full parsing of the JSON data and constructs a new CustomBiome instance. Complexity is proportional to the size (N) of the input JSON definition. |

## Integration Patterns

### Standard Usage
This class should only be invoked by the core world generation systems responsible for loading game assets. The typical pattern involves creating a loader, executing the load operation once, and then storing the resulting biome in a central registry.

```java
// Executed within a central asset loading system
BiomeFileContext context = getBiomeContextFor("my_custom_forest");
JsonElement biomeJson = parseBiomeFile("my_custom_forest.json");
Biome[] availableTileBiomes = biomeRegistry.getTileBiomes();
SeedString worldgenSeed = world.getSeed();

// Instantiate, use once, and discard
CustomBiomeJsonLoader loader = new CustomBiomeJsonLoader(
    worldgenSeed,
    assetPath,
    biomeJson,
    context,
    availableTileBiomes
);
CustomBiome biome = loader.load();

// The resulting object is now used by the world generator
biomeRegistry.register(biome);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain a reference to a CustomBiomeJsonLoader instance to call `load` multiple times. Each call to `load` is not guaranteed to be idempotent and the loader is designed as a single-shot factory.

- **Incorrect Context:** Supplying an incomplete or mismatched `tileBiomes` array is a critical error. If the JSON definition references a tile biome not present in the array, world generation will fail with a runtime exception.

- **Manual Instantiation in Game Logic:** Game logic systems (e.g., entity behaviors, block updates) must never instantiate loaders directly. Biomes should always be retrieved from a central, authoritative registry.

## Data Pipeline
The CustomBiomeJsonLoader is a key transformation step in the world generation asset pipeline. It converts declarative data into an executable, object-oriented representation.

> Flow:
> Biome JSON File on Disk -> GSON Parser -> `JsonElement` -> **CustomBiomeJsonLoader** -> `CustomBiome` Object -> World Generator Biome Registry

