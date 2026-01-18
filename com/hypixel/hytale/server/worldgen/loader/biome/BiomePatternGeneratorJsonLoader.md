---
description: Architectural reference for BiomePatternGeneratorJsonLoader
---

# BiomePatternGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.biome
**Type:** Transient

## Definition
```java
// Signature
public class BiomePatternGeneratorJsonLoader extends JsonLoader<SeedStringResource, BiomePatternGenerator> {
```

## Architecture & Concepts

The BiomePatternGeneratorJsonLoader is a specialized factory class within the server's world generation framework. Its primary function is to deserialize a JSON configuration object into a fully initialized BiomePatternGenerator instance. This class acts as a critical bridge between the declarative world generation data files and the procedural code that actually builds the world.

The most significant architectural pattern employed here is the **resolution of a circular dependency** through a proxy-like provider. A core challenge in biome generation is that the shape and placement of biomes (determined by a point generator) can depend on the properties of the biomes themselves (e.g., a "large" biome should influence the point grid). This creates a "chicken-and-egg" problem: the generator needs the biome properties to be created, but the biome properties can only be queried from a fully-created generator.

This class solves this by:
1.  Creating a proxy object, BiomePatternGeneratorSizeModifierProvider, and passing it to the underlying PointGeneratorJsonLoader.
2.  The PointGeneratorJsonLoader decorates its distance calculation logic using this provider.
3.  The BiomePatternGenerator is then instantiated using the now-configured point generator.
4.  Finally, the fully-built BiomePatternGenerator instance is injected back into the proxy provider, completing the dependency loop.

This allows the point generation logic to lazily query biome properties from the final generator object during its calculations, even though that object did not exist when the point generator itself was being constructed.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level world generation orchestrator during the server's world data loading phase. It is provided with all necessary context, including the world seed, a reference to the data folder, the specific JSON element to parse, and the registries of available TileBiomes and CustomBiomes.
-   **Scope:** The instance has a very short lifetime, existing only for the duration of the `load` method call. It is a single-use, transient factory.
-   **Destruction:** The loader object is eligible for garbage collection immediately after the `load` method returns its result. It holds no persistent state and is not registered in any service locator.

## Internal State & Concurrency
-   **State:** The class holds immutable references to its constructor arguments. Its internal state is considered constant for its entire lifecycle. It does not perform any caching.
-   **Thread Safety:** This class is **not thread-safe** and must not be used concurrently. The `load` method orchestrates a complex, stateful initialization sequence, including the late-binding of the BiomePatternGenerator to its provider. The entire world generation loading pipeline is designed to be a single-threaded, synchronous process.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BiomePatternGeneratorJsonLoader(...) | constructor | O(1) | Constructs the loader with all necessary worldgen context. |
| load() | BiomePatternGenerator | O(N) | The primary factory method. Parses the JSON, resolves internal dependencies, and returns a fully configured generator. N corresponds to the complexity of the GridGenerator JSON configuration. |

## Integration Patterns

### Standard Usage

This class is not intended for direct use by game logic developers. It is invoked by the server's core world generation loading system. The standard pattern is to instantiate, load, and immediately discard the loader.

```java
// Invoked within a higher-level world configuration loader
JsonElement biomePatternJson = worldConfig.get("biomePattern");
IWeightedMap<TileBiome> tileBiomes = biomeRegistry.getTileBiomes();
CustomBiome[] customBiomes = biomeRegistry.getCustomBiomes();

BiomePatternGeneratorJsonLoader loader = new BiomePatternGeneratorJsonLoader(
    worldSeed,
    dataPath,
    biomePatternJson,
    tileBiomes,
    customBiomes
);

// The generator is the final product; the loader is no longer needed.
BiomePatternGenerator generator = loader.load();
context.registerService(generator);
```

### Anti-Patterns (Do NOT do this)
-   **Instance Reuse:** Do not retain an instance of this loader to call `load` multiple times. It is designed for one-shot creation. Re-running `load` will produce a distinct but functionally identical object graph, which is inefficient.
-   **Concurrent Access:** Never share a loader instance across multiple threads. The internal state management during the `load` process is not synchronized and will result in unpredictable behavior and likely runtime exceptions.
-   **Incomplete Context:** Instantiating the loader with null or empty biome maps will cause NullPointerExceptions during the `load` operation, as the underlying generator logic depends on this data.

## Data Pipeline

This component functions within a configuration-time data pipeline, transforming a static data format into an active in-memory object.

> Flow:
> Worldgen JSON File -> GSON Parser -> `JsonElement` -> **BiomePatternGeneratorJsonLoader** -> `BiomePatternGenerator` Instance -> World Generation Engine

