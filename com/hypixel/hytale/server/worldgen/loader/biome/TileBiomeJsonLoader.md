---
description: Architectural reference for TileBiomeJsonLoader
---

# TileBiomeJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.biome
**Type:** Transient

## Definition
```java
// Signature
public class TileBiomeJsonLoader extends BiomeJsonLoader {
```

## Architecture & Concepts
The TileBiomeJsonLoader is a specialized component within the server's world generation framework. It acts as a deserializer and factory, responsible for translating a specific JSON configuration file into a fully hydrated, in-memory TileBiome object.

This class extends the more generic BiomeJsonLoader, inheriting the logic for parsing common biome properties like layers, prefabs, and environmental settings. Its primary responsibility is to handle the properties unique to *tile-based* biomes, which are discrete, often smaller biomes used in a tile-based generation system. These unique properties include **Weight** and **SizeModifier**, which influence the biome's placement and scale during world generation.

A critical architectural feature is its interaction with the procedural seeding system. The constructor appends a biome-specific suffix to the world seed. This creates a deterministic, hierarchical seed chain, ensuring that any procedural elements loaded for this biome (such as noise functions) are unique to this biome type but are still deterministically derived from the main world seed.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level orchestration service, such as a BiomeRegistryLoader. This occurs when the service scans the server's data files and identifies a JSON file defining a tile biome. The orchestrator is responsible for providing all constructor dependencies, including the raw JSON data and a BiomeFileContext.
- **Scope:** Extremely short-lived. The object's lifespan is typically confined to a single method call. It is created, its `load` method is invoked once, and the resulting TileBiome object is returned. After this, the loader instance has no further purpose and is eligible for garbage collection.
- **Destruction:** Managed entirely by the Java garbage collector. There are no manual cleanup or disposal methods required.

## Internal State & Concurrency
- **State:** The TileBiomeJsonLoader is stateful but its state is effectively immutable after construction. It holds references to the JSON data, the biome context, and the data folder path, but these are only used as inputs for the `load` operation. It performs no internal caching or state modification across multiple calls.
- **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded execution. Sharing a single instance across multiple threads and calling `load` concurrently will result in undefined behavior. However, the broader system can achieve parallelism by creating separate TileBiomeJsonLoader instances on different threads to load different biome files simultaneously.

## API Surface
The public contract is minimal, exposing only the functionality required to construct the loader and execute the loading process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | TileBiome | O(N) | Parses the internal JSON data and constructs a new TileBiome instance. Complexity is proportional to the number of properties defined in the biome's JSON file. |

## Integration Patterns

### Standard Usage
The loader should be treated as a single-use factory. A central registry or manager instantiates it with the necessary context, calls `load` to get the final object, and then discards the loader instance.

```java
// Invoked by a higher-level system like a BiomeRegistry
BiomeFileContext context = createBiomeContextFor("my_biome.json");
SeedString<SeedStringResource> worldSeed = server.getWorldSeed();
JsonElement biomeJsonData = assetManager.loadJson("my_biome.json");
Path dataPath = server.getDataFolderPath();

// Create, use, and discard
TileBiomeJsonLoader loader = new TileBiomeJsonLoader(worldSeed, dataPath, biomeJsonData, context);
TileBiome biome = loader.load();

// The loaded biome is now ready for registration and use
biomeRegistry.register(biome);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain a reference to a loader instance to call `load` multiple times. Each loader is configured for a single, specific JSON source.
- **State Manipulation:** Do not attempt to modify the loader's internal state via reflection after construction. The object's integrity depends on the state provided at creation.
- **Concurrent Access:** Do not share a single loader instance across multiple threads. This can lead to race conditions, especially if the underlying JSON libraries or parent class methods are not designed for concurrent use.

## Data Pipeline
This class is a key transformation step in the server's configuration-to-runtime data pipeline. It converts declarative data (JSON) into an active, operational game object (TileBiome).

> Flow:
> Biome JSON File on Disk -> Asset Loading System -> Raw JsonElement -> **TileBiomeJsonLoader** -> In-Memory TileBiome Object -> World Generator Biome Registry

