---
description: Architectural reference for ClimateGridJsonLoader
---

# ClimateGridJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.climate
**Type:** Transient

## Definition
```java
// Signature
public class ClimateGridJsonLoader<K extends SeedResource> extends JsonLoader<K, ClimateNoise.Grid> {
```

## Architecture & Concepts
The ClimateGridJsonLoader is a specialized, single-purpose parser responsible for translating a JSON configuration object into a fully materialized ClimateNoise.Grid instance. It serves as a critical bridge between the declarative world generation data files and the procedural generation engine.

This class embodies a "configuration-over-code" design philosophy. It allows world designers to define and tweak the fundamental properties of a climate grid—such as its scale, randomness, and point distribution—without modifying core engine code. It operates as a delegate for a higher-level configuration loader, which isolates the logic for parsing this specific schema.

A key architectural feature is its sophisticated seeding mechanism. The loader combines a base world seed (passed via the SeedString object) with an optional, more specific seed from the JSON data. This enables a layered seeding strategy, where a global seed can produce a deterministic world, while specific components within that world can be further varied in a repeatable manner.

The loader's output, a ClimateNoise.Grid, is a foundational data structure used by the climate system to determine biome placement and other large-scale environmental features.

## Lifecycle & Ownership
- **Creation:** An instance of ClimateGridJsonLoader is created by a parent configuration system when it encounters a JSON object that represents a climate grid. It is provided with all necessary context at instantiation, including the JSON data, a base seed, and a path to the data folder.

- **Scope:** This object is ephemeral and has a very short lifecycle. Its existence is scoped exclusively to the duration of the parsing operation for a single climate grid definition.

- **Destruction:** The loader is intended to be used once to call the **load** method. After the resulting ClimateNoise.Grid object is returned, the loader instance has no further purpose and becomes eligible for garbage collection. The ownership of the *produced grid* is transferred to the calling system, not the loader.

## Internal State & Concurrency
- **State:** The internal state of the loader (seed, dataFolder, json) is provided during construction and is not modified thereafter. The object is effectively **immutable**. The **load** method is a pure function that produces an output based solely on this initial state.

- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable state and the absence of side effects, a single instance can be safely used by multiple threads, although the standard usage pattern involves creating a new instance for each parsing task.

## API Surface
The public contract is minimal, exposing only the functionality required to produce a grid object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ClimateNoise.Grid | O(1) | Parses the internal JSON data and constructs a new ClimateNoise.Grid. Returns a static default grid if no JSON was provided. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most systems. It is designed to be invoked by a higher-level loader that manages world generation assets.

```java
// Hypothetical usage within a parent worldgen loader
JsonElement gridConfig = worldConfigFile.get("climateGridDefinition");
SeedString worldSeed = context.getWorldSeed();

// Delegate the specific parsing task to the specialized loader
ClimateGridJsonLoader<SeedResource> loader = new ClimateGridJsonLoader<>(worldSeed, dataPath, gridConfig);
ClimateNoise.Grid climateGrid = loader.load();

// The resulting grid is now used by the climate engine
climateEngine.registerGrid(climateGrid);
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to modify the loader's state after creation via reflection or other means. Its behavior is only guaranteed with the state provided at construction.
- **Long-Term Storage:** Do not hold references to ClimateGridJsonLoader instances in caches or long-lived services. They are lightweight, transient objects designed to be created and discarded. Storing them provides no benefit and prevents garbage collection.
- **Instantiation Without JSON:** Creating an instance with a null JsonElement is a valid fallback, but it should only be used as a planned default. Relying on this behavior for primary logic indicates a flaw in the upstream data pipeline.

## Data Pipeline
The ClimateGridJsonLoader acts as a transformation step in the world generation asset pipeline. It consumes a raw data structure (JSON) and produces a specialized, in-memory engine object (Grid).

> Flow:
> World Generation JSON File -> Parent Asset Parser -> JsonElement -> **ClimateGridJsonLoader** -> ClimateNoise.Grid Object -> Climate Engine

