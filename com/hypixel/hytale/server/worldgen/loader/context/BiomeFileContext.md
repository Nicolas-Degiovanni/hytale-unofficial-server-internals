---
description: Architectural reference for BiomeFileContext
---

# BiomeFileContext

**Package:** com.hypixel.hytale.server.worldgen.loader.context
**Type:** Transient

## Definition
```java
// Signature
public class BiomeFileContext extends FileContext<ZoneFileContext> {
```

## Architecture & Concepts
The **BiomeFileContext** is a specialized, immutable data-holding class that represents a single biome definition file discovered on the filesystem. It functions as a node in a hierarchical tree of world generation assets, with its direct parent being a **ZoneFileContext**.

Its primary architectural role is to act as a metadata container during the server's world generation asset loading phase. It does not contain the biome's actual data (e.g., block palettes, noise parameters). Instead, it encapsulates the essential identifiers required to locate, load, and classify that data: its unique ID, name, filesystem path, and, most critically, its **Type**.

The nested **Type** enum (**Tile**, **Custom**) is a key design element. It allows the world generation loader to differentiate between standard, grid-based biomes and more specialized, procedurally unique biomes by inspecting the file's naming convention. This classification, performed by the static **getBiomeType** method, is the first step in the pipeline for parsing and integrating a biome into the world generator.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level world generation asset scanner or loader. This process typically involves walking a directory tree, identifying potential biome files, classifying them using **getBiomeType**, and then constructing a **BiomeFileContext** instance. Ownership is held by the loader that created it.
- **Scope:** The object's lifetime is transient, scoped to the asset loading and validation process. It exists in memory while the server is building its registry of available world generation assets.
- **Destruction:** Once the biome's data has been fully parsed and loaded into a more permanent, runtime-optimized data structure, the **BiomeFileContext** is no longer referenced and becomes eligible for garbage collection. It does not persist for the entire server session.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields, including the parent reference and the biome type, are assigned exclusively during construction and are declared final. This object is a snapshot of file metadata at a specific point in time.
- **Thread Safety:** **Fully thread-safe**. Due to its immutability, an instance of **BiomeFileContext** can be safely shared and read across multiple threads without any need for external synchronization or locks. The static utility method **getBiomeType** is also stateless and thread-safe.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | BiomeFileContext.Type | O(1) | Returns the pre-determined biome type (**Tile** or **Custom**). |
| getBiomeType(Path path) | static BiomeFileContext.Type | O(1) | Classifies a file path as a specific biome type based on its filename prefix. Throws an Error if the filename does not match a known pattern. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation by game logic developers. It is used internally by the world generation asset loading system. A typical interaction involves using the static classifier before creating an instance.

```java
// A worldgen asset loader discovers a file
Path biomeFilePath = Paths.get("worldgen/zones/forest/Tile.forest_floor.json");
ZoneFileContext parentZone = getParentZoneContext(); // Assume this exists

try {
    // 1. Classify the file using the static utility method
    BiomeFileContext.Type type = BiomeFileContext.getBiomeType(biomeFilePath);

    // 2. If successful, create the context object
    BiomeFileContext biomeContext = new BiomeFileContext(
        generateId(),
        "forest_floor",
        biomeFilePath,
        type,
        parentZone
    );

    // 3. Add the context to an asset registry for later processing
    assetRegistry.registerBiome(biomeContext);

} catch (Error e) {
    // The file is not a valid biome definition; log and skip.
    log.warn("Skipping invalid file: " + biomeFilePath, e);
}
```

### Anti-Patterns (Do NOT do this)
- **Catching Error:** The **getBiomeType** method throws an **Error**, not an **Exception**. This indicates a critical problem with the asset files, such as a malformed filename. Code should not attempt to catch this; it signals a configuration or asset packaging issue that must be fixed by a developer.
- **State Assumption:** Do not assume the file pointed to by the context's path still exists or that its contents are valid. This class only represents the metadata at the time of discovery. Subsequent I/O operations are required to read the file and are subject to failure.
- **Manual Instantiation:** Avoid creating instances of **BiomeFileContext** manually. They should only be created by the authoritative asset loading system to ensure the integrity of the world generation asset tree.

## Data Pipeline
The **BiomeFileContext** is an early-stage component in the data flow that transforms on-disk asset files into in-memory, usable world generation rules.

> Flow:
> Filesystem Directory Scan -> Raw **Path** Object -> **BiomeFileContext.getBiomeType** (Classification) -> **BiomeFileContext** (Instantiation) -> WorldGen Asset Registry -> Biome Data Parser -> Runtime Biome Object

