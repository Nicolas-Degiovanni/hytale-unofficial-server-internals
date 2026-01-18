---
description: Architectural reference for ClimateNoiseJsonLoader
---

# ClimateNoiseJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.climate
**Type:** Transient / Factory

## Definition
```java
// Signature
public class ClimateNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, ClimateNoise> {
```

## Architecture & Concepts
The ClimateNoiseJsonLoader is a specialized factory class responsible for deserializing a JSON configuration into a fully-realized ClimateNoise object. It acts as a high-level **Orchestrator** within the data-driven world generation system.

Its primary architectural role is not to parse raw data itself, but to delegate the parsing of specific, complex JSON sub-objects to other, more specialized loaders. For example, it delegates the loading of noise functions to NoisePropertyJsonLoader and grid definitions to ClimateGridJsonLoader. This **Composition over Inheritance** pattern makes the loading process modular and extensible.

This class forms a critical bridge between the static, declarative world generation assets (JSON files) and the dynamic, in-memory objects used by the procedural generation engine at runtime. The generic parameter K, constrained to SeedResource, ensures that all loaded noise properties are correctly seeded and deterministic for a given world.

### Lifecycle & Ownership
-   **Creation:** An instance of ClimateNoiseJsonLoader is created on-demand by a higher-level configuration manager when it needs to process a specific climate definition from a JSON file. It is not a singleton and is not managed by a dependency injection container.
-   **Scope:** The object's lifetime is exceptionally short. It exists only for the duration of the `load` method call. Once the resulting ClimateNoise object is returned, the loader instance has served its purpose and is immediately eligible for garbage collection.
-   **Destruction:** Cleanup is handled automatically by the Java garbage collector. There are no native resources or persistent connections to manage.

## Internal State & Concurrency
-   **State:** The loader's state consists of the seed, data folder path, and the root JsonElement, all of which are provided at construction. This state is treated as **immutable** for the object's lifetime. The class is stateless concerning its operational logic; multiple calls to `load` on the same instance would produce identical objects but would be an inefficient use of the pattern.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to operate within a single-threaded asset loading or world initialization pipeline. Concurrent access to the `load` method would result in undefined behavior, as the underlying JsonElement and delegated loaders are not guaranteed to be thread-safe.

## API Surface
The public contract is minimal, consisting of the constructor for instantiation and the `load` method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ClimateNoise | O(N) | Orchestrates the full deserialization process, where N is the number of nodes in the JSON object. Delegates to sub-loaders and assembles the final ClimateNoise object. Throws if required JSON keys are missing. |

## Integration Patterns

### Standard Usage
The loader is intended to be instantiated with a specific JSON context and used immediately to produce a single object.

```java
// Assume 'climateJsonElement' is a JsonElement parsed from a file
// and 'worldSeed' is a valid SeedString instance.

// 1. Instantiate the loader with the specific JSON context
ClimateNoiseJsonLoader loader = new ClimateNoiseJsonLoader<>(worldSeed, dataPath, climateJsonElement);

// 2. Execute the load operation to get the final object
ClimateNoise climateNoise = loader.load();

// 3. Use the resulting object in the world generator
worldGenerator.setClimateNoise(climateNoise);

// The 'loader' instance is no longer needed and can be discarded.
```

### Anti-Patterns (Do NOT do this)
-   **Instance Caching:** Do not cache and reuse instances of ClimateNoiseJsonLoader. Each loader is tied to the specific JsonElement passed to its constructor. Create a new instance for each distinct configuration you need to load.
-   **Concurrent Loading:** Do not call the `load` method from multiple threads on a single instance. The internal state and the delegated loaders are not designed for concurrent access.
-   **Direct Sub-Loader Invocation:** Avoid calling the protected `loadGrid`, `loadContinentNoise`, etc., methods directly. The public `load` method guarantees the correct order of operations and object assembly. Bypassing it can lead to partially constructed or invalid ClimateNoise objects.

## Data Pipeline
This class acts as a central hub in the data transformation pipeline, converting a raw JSON structure into a complex, usable game object.

> Flow:
> `climate.json` -> **Gson Parser** -> `JsonElement` -> **ClimateNoiseJsonLoader**
> 
> **ClimateNoiseJsonLoader** then delegates:
> -   `"Grid": {}` -> `ClimateGridJsonLoader` -> `ClimateNoise.Grid`
> -   `"Continent": {}` -> `NoisePropertyJsonLoader` -> `NoiseProperty`
> -   `"Temperature": {}` -> `NoisePropertyJsonLoader` -> `NoiseProperty`
> -   `"Intensity": {}` -> `NoisePropertyJsonLoader` -> `NoiseProperty`
> -   `"Thresholds": {}` -> `ContinentThresholdsJsonLoader` -> `ClimateNoise.Thresholds`
> 
> Finally, **ClimateNoiseJsonLoader** assembles these components into a single `ClimateNoise` object, which is passed to the **World Generation Engine**.

