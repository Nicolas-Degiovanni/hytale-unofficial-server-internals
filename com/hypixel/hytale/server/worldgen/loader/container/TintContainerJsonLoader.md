---
description: Architectural reference for TintContainerJsonLoader
---

# TintContainerJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.container
**Type:** Transient Factory

## Definition
```java
// Signature
public class TintContainerJsonLoader extends JsonLoader<SeedStringResource, TintContainer> {
```

## Architecture & Concepts
The TintContainerJsonLoader is a specialized deserializer within the server-side World Generation pipeline. Its primary function is to translate a structured JSON definition into a fully-realized, in-memory TintContainer object. This object dictates the procedural coloring rules for world features, such as biomes, blocks, or foliage.

This class operates as a concrete implementation of the abstract JsonLoader, adhering to a schema that defines default colors and a list of conditional, noise-driven color entries. It is a critical bridge between static data assets (JSON files) and the dynamic, procedural systems that build the game world.

A key architectural pattern employed is **Composition over Inheritance**. Instead of containing all parsing logic, TintContainerJsonLoader delegates the responsibility of parsing complex, nested JSON objects to other specialized loaders, such as NoisePropertyJsonLoader and NoiseMaskConditionJsonLoader. This maintains a clean separation of concerns, making the system more modular and maintainable.

The propagation of the SeedString is fundamental to its design. The loader systematically appends unique suffixes to the incoming seed as it descends into the JSON tree (e.g., for each entry in an array). This ensures that all procedural elements, like noise functions, are initialized with a deterministic and unique seed, which is a cornerstone of reproducible world generation.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level asset loading service when a JSON file containing a tinting definition needs to be parsed. It is never managed as a singleton or shared resource.
- **Scope:** The object's lifetime is ephemeral, strictly confined to the duration of a single `load` operation. It is a single-use tool.
- **Destruction:** Once the `load` method returns the constructed TintContainer, the loader instance has served its purpose and becomes eligible for garbage collection. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state of the loader (seed, data folder, and the source JSON) is provided at construction and is treated as immutable. The `load` method is a pure function of this initial state and does not modify it. The class does not cache any data.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution within a larger asset loading process. Each distinct loading task must create its own TintContainerJsonLoader instance to prevent race conditions.

## API Surface
The public contract is minimal, exposing only the functionality required to perform the load operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | TintContainer | O(N) | Parses the source JSON and constructs a new TintContainer. N is the number of entries in the "Entries" array. Throws unchecked exceptions on malformed data or schema violations. |

## Integration Patterns

### Standard Usage
The loader is invoked by a parent system responsible for loading a larger asset, such as a biome definition. The parent extracts the relevant JSON sub-tree and passes it to the loader along with a context-specific seed.

```java
// A higher-level service loads a biome's JSON definition.
// It extracts the "tints" object to be processed.

JsonElement tintJson = biomeDefinition.get("tints");
SeedString biomeSeed = worldContext.getSeed().append(".forestBiome");
Path dataFolder = worldContext.getDataFolder();

// A new loader is created for this specific, one-shot task.
TintContainerJsonLoader loader = new TintContainerJsonLoader(biomeSeed, dataFolder, tintJson);
TintContainer tintContainer = loader.load();

// The resulting object is now integrated into the biome's runtime properties.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain a reference to a loader instance to use it for multiple JSON objects. Its internal state is tied to the JSON provided at construction. Create a new loader for each distinct asset.
- **Generic Seeding:** Do not instantiate the loader with a generic or null SeedString. This will break the determinism of the world generator, leading to unpredictable and inconsistent procedural noise across different assets.
- **Exception Swallowing:** The `load` method will throw runtime exceptions like IllegalArgumentException for invalid data. These exceptions must be caught and handled by the asset loading framework to prevent a server crash and to provide actionable error logs for content designers.

## Data Pipeline
The class functions as a transformation step in the broader asset-to-engine data flow.

> Flow:
> JSON File on Disk → Asset Manager → `JsonElement` → **TintContainerJsonLoader** → `TintContainer` Object → World Generation Engine

