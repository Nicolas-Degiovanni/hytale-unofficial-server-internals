---
description: Architectural reference for CustomBiomeGeneratorJsonLoader
---

# CustomBiomeGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.biome
**Type:** Transient

## Definition
```java
// Signature
public class CustomBiomeGeneratorJsonLoader extends JsonLoader<SeedStringResource, CustomBiomeGenerator> {
```

## Architecture & Concepts

The CustomBiomeGeneratorJsonLoader is a specialized data loader that operates within the server-side world generation pipeline. Its primary function is to act as a **translator**, deserializing a specific JSON configuration structure into a fully realized, in-memory CustomBiomeGenerator object.

This class is a cornerstone of Hytale's data-driven design philosophy for world generation. It allows designers and developers to define complex biome placement rules—based on noise maps, biome masks, and priority—in external JSON assets rather than hard-coding them in Java.

Architecturally, this loader functions as a **composite builder**. It does not parse the entire structure itself; instead, it orchestrates a set of more granular loaders, such as NoisePropertyJsonLoader and DoubleThresholdJsonLoader, to process nested sections of the JSON data. This compositional approach promotes separation of concerns, making the loading process for complex objects more modular and maintainable.

## Lifecycle & Ownership

-   **Creation:** An instance of CustomBiomeGeneratorJsonLoader is created by a higher-level orchestrator within the world generation system, likely a master BiomeLoader or a similar factory. This occurs when the system processes a biome asset file and identifies a section that defines a custom generator. All dependencies, including the raw JsonElement, the current BiomeFileContext, and the list of available tileBiomes, are injected via the constructor.

-   **Scope:** The lifetime of a CustomBiomeGeneratorJsonLoader instance is extremely short and task-specific. It is designed to be used for a single `load` operation. Its scope is confined to the method call that requires the CustomBiomeGenerator object.

-   **Destruction:** The object is eligible for garbage collection immediately after the `load` method returns its result and the reference to the loader is dropped. It manages no persistent state or external resources that would require an explicit destruction or cleanup phase.

## Internal State & Concurrency

-   **State:** The internal state of the loader is established at construction and is treated as **effectively immutable**. Fields such as biomeContext, tileBiomes, and the source JsonElement are set once and are not modified during the object's lifecycle. The class is stateful only in that it holds this configuration for the duration of the `load` operation.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed to operate within the single-threaded asset loading phase of world generation. Internal operations, such as the creation of a HashMap in `generateNameBiomeMapping`, are not synchronized.

    **WARNING:** Sharing an instance of this loader across multiple threads will lead to unpredictable behavior and potential race conditions. The world generation framework guarantees single-threaded access during its initialization phase.

## API Surface

The public contract is minimal, exposing only the primary `load` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | CustomBiomeGenerator | O(N) | Deserializes the configured JSON into a CustomBiomeGenerator object. N is the number of biome names in the BiomeMask array. Throws runtime exceptions on malformed data. |

## Integration Patterns

### Standard Usage

This loader is not intended for direct use by most developers. It is invoked by the world generation framework. The standard pattern involves the framework creating an instance with the necessary context and immediately calling `load` to retrieve the configured generator.

```java
// Pseudo-code illustrating framework-level usage
JsonElement generatorJson = parseBiomeFile("my_biome.json");
Biome[] availableBiomes = biomeRegistry.getAllBiomes();
BiomeFileContext context = new BiomeFileContext(...);

// The framework instantiates the loader with all necessary context
CustomBiomeGeneratorJsonLoader loader = new CustomBiomeGeneratorJsonLoader(
    seed, dataFolderPath, generatorJson, context, availableBiomes
);

// The loader is used once to produce the final object
CustomBiomeGenerator generator = loader.load();

// The generator is now ready to be used by the world generator
worldGenerator.addCustomGenerator(generator);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Avoid `new CustomBiomeGeneratorJsonLoader()` in game logic. The loader requires a precise and valid context (BiomeFileContext, a complete list of tileBiomes) that is only guaranteed to be correct when provided by the world generation framework during its loading phase. Manual instantiation risks creating invalid or improperly configured generators.

-   **Instance Caching or Re-use:** Do not cache and re-use instances of this loader. It is a cheap, short-lived object. Caching provides no performance benefit and increases the risk of using stale context if the underlying assets were to be reloaded.

-   **Modification via Reflection:** The internal state is not designed to be modified after construction. Altering its fields via reflection will result in undefined behavior and likely cause critical failures during world generation.

## Data Pipeline

The CustomBiomeGeneratorJsonLoader is a critical transformation step in the data pipeline that turns declarative asset files into executable world generation logic.

> Flow:
> Biome JSON Asset on Disk -> JsonElement (in memory) -> **CustomBiomeGeneratorJsonLoader** -> CustomBiomeGenerator (runtime object) -> World Generation Engine

