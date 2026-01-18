---
description: Architectural reference for WaterContainerJsonLoader
---

# WaterContainerJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.container
**Type:** Transient / Factory

## Definition
```java
// Signature
public class WaterContainerJsonLoader extends JsonLoader<SeedStringResource, WaterContainer> {
```

## Architecture & Concepts

The WaterContainerJsonLoader is a specialized factory class responsible for deserializing a JSON configuration into a fully realized WaterContainer object. It serves as a critical bridge between the declarative, file-based world generation assets and the procedural, in-memory systems that construct the game world.

This loader is a component within a larger, compositional system of JSON loaders. It does not operate in isolation; it delegates the parsing of complex sub-properties—such as noise functions, coordinate conditions, and value ranges—to other dedicated loaders like NoisePropertyJsonLoader and DoubleRangeJsonLoader. This compositional design promotes separation of concerns and allows for complex, nested procedural definitions within the JSON assets.

A core architectural concept is the hierarchical propagation of the SeedString. The initial seed is appended with contextual identifiers (e.g., ".WaterContainer", "-0", ".Entry") as the loader descends into the JSON structure. This ensures that every procedurally generated component, such as a noise function, is derived from a unique and deterministic seed, guaranteeing reproducible world generation.

The loader supports two primary JSON schemas for defining water:
1.  **Simple Schema:** A top-level object defining a single water type via a Block or Fluid key, with a uniform height definition. This is an optimized path for common cases.
2.  **Complex Schema:** A top-level object containing an Entries array, where each element defines a distinct layer or type of water with its own set of conditions, materials, and procedural height suppliers.

## Lifecycle & Ownership

-   **Creation:** An instance is created on-demand by a higher-level world generation process, such as a biome or zone loader, whenever it encounters a JSON object that defines a water container. It is **not** a singleton or a managed service.
-   **Scope:** The object's lifetime is exceptionally short. It is designed to be single-use, existing only for the duration of the call to its load method.
-   **Destruction:** The loader instance becomes eligible for garbage collection immediately after the load method returns its WaterContainer result. It holds no persistent state and is not retained by any system.

## Internal State & Concurrency

-   **State:** The internal state of the loader (seed, dataFolder, json) is provided at construction and is treated as immutable. The class is fundamentally a read-only parser of its input JSON. It performs no caching or state modification during its operation.
-   **Thread Safety:** This class is **not thread-safe** for concurrent use on a single instance. The load method is not designed to be re-entrant. However, the transient nature of the loader makes it inherently safe for parallel world generation. Each worker thread should create its own distinct instance of WaterContainerJsonLoader to parse its assigned data, preventing any data races.

## API Surface

The public contract is minimal, exposing only the core factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | WaterContainer | O(N) | Deserializes the provided JSON into a WaterContainer object. N is the number of entries in the JSON. Throws Error on asset lookup failure or severe schema violations. |

## Integration Patterns

### Standard Usage

This loader is intended to be instantiated and used immediately by a parent system responsible for parsing a larger configuration file.

```java
// A parent loader receives a JsonElement for a water definition
JsonElement waterJson = biomeDefinition.get("water");
SeedString biomeSeed = ...;
Path dataFolder = ...;

// Create a transient loader, call load(), and use the result
WaterContainer container = new WaterContainerJsonLoader(biomeSeed, dataFolder, waterJson).load();

// The loader instance is now discarded
worldChunk.applyWater(container);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Caching:** Do not retain and reuse a WaterContainerJsonLoader instance. It is configured for a specific JSON element and is not designed for multiple invocations.
-   **Ignoring Errors:** The loader can throw a Java Error, not an Exception, for critical configuration issues like a missing BlockType or Fluid asset. This indicates a fatal problem with the game's assets. This must be caught by a high-level error handler in the world generation pipeline to prevent a server crash.
-   **State Modification:** Do not attempt to modify the loader's state via reflection. The object's integrity depends on its immutable inputs.

## Data Pipeline

The WaterContainerJsonLoader functions as a specific step in the data transformation pipeline that turns static asset files into a live, interactive world.

> Flow:
> World Zone JSON File -> Biome Loader -> JsonElement (for water) -> **WaterContainerJsonLoader** -> WaterContainer Object -> Procedural World Generator -> Final Chunk Data

