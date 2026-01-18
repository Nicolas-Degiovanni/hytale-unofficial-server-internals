---
description: Architectural reference for ZoneCustomBiomesJsonLoader
---

# ZoneCustomBiomesJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.zone
**Type:** Transient

## Definition
```java
// Signature
public class ZoneCustomBiomesJsonLoader extends JsonLoader<SeedStringResource, CustomBiome[]> {
```

## Architecture & Concepts
The ZoneCustomBiomesJsonLoader is a specialized, transient worker class within the server's world generation pipeline. Its sole responsibility is to orchestrate the loading of all *custom biome* definitions associated with a single parent *zone*.

This class acts as a bridge between the file system configuration (JSON files defining biomes) and the in-memory object representation (CustomBiome instances) used by the procedural generation engine. It does not parse the biome files itself; instead, it demonstrates a clear separation of concerns by delegating the parsing of each individual biome file to a more granular loader, the CustomBiomeJsonLoader.

Its primary architectural role is to aggregate and prepare a complete, sorted set of custom biomes required for a specific zone's generation logic. The final output is sorted by a priority value, ensuring that biome placement rules can be evaluated in a deterministic and predictable order.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor (`new`) by a higher-level component, typically a ZoneLoader, during the bootstrap phase of a single zone. It receives all necessary context, including file paths and existing biome data, upon creation.
- **Scope:** The object's lifetime is extremely short. It is designed to be single-use and exists only for the duration of the `load` method call. Once it returns the `CustomBiome[]` array, it serves no further purpose and is immediately eligible for garbage collection.
- **Destruction:** Managed by the standard Java garbage collector. No explicit cleanup methods are required or provided.

## Internal State & Concurrency
- **State:** The class is stateful, holding references to the ZoneFileContext, tileBiomes, and other configuration data passed into its constructor. This state is treated as immutable for the object's lifetime and is used exclusively as input for the `load` method.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be confined to the single thread responsible for world generation. Internal operations, such as iterating and populating an array using an index, are inherently unsafe for concurrent access.

**WARNING:** Do not share instances of this class across threads or invoke its methods concurrently. All world generation loading must occur in a serialized, single-threaded manner.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | CustomBiome[] | O(N * M) | Orchestrates the loading process. Iterates through N custom biomes, reading and parsing M-sized files for each. Returns a complete array sorted by priority. Throws an Error on I/O or parsing failure. |

## Integration Patterns

### Standard Usage
This class is not retrieved from a service registry. It is intended to be instantiated directly by a parent loader that manages the construction of a zone.

```java
// Context: Inside a higher-level ZoneLoader
ZoneFileContext zoneContext = getZoneContext();
Biome[] availableTileBiomes = getTileBiomes();
SeedString<SeedStringResource> worldSeed = getSeed();
Path dataRoot = getDataFolderPath();

// Instantiate the loader with all required context
ZoneCustomBiomesJsonLoader loader = new ZoneCustomBiomesJsonLoader(
    worldSeed,
    dataRoot,
    zoneJsonDefinition,
    zoneContext,
    availableTileBiomes
);

// Execute the single-use load operation
CustomBiome[] customBiomes = loader.load();

// The loader instance is now discarded and customBiomes are used
// to configure the zone's world generation behavior.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain a reference to this loader after the `load` method has completed. It is a single-shot worker and holds context that may become stale.
- **Concurrent Loading:** Do not attempt to parallelize the loading process by calling `load` from multiple threads. This will result in unpredictable behavior and likely cause a crash.
- **Incomplete Context:** Creating an instance with a null or partially populated ZoneFileContext will lead to a NullPointerException or an Error during the `load` operation. Ensure all dependencies are fully resolved before instantiation.

## Data Pipeline
This class executes a data transformation pipeline, converting file paths into a structured, in-memory array of game objects.

> Flow:
> ZoneFileContext (list of biome file paths) -> **ZoneCustomBiomesJsonLoader** -> Iterates and reads each `.json` file from disk -> Delegates parsing to `CustomBiomeJsonLoader` -> Collects `CustomBiome` instances -> Sorts array by priority -> `CustomBiome[]` (Final Output)

