---
description: Architectural reference for ZoneFileContext
---

# ZoneFileContext

**Package:** com.hypixel.hytale.server.worldgen.loader.context
**Type:** Transient

## Definition
```java
// Signature
public class ZoneFileContext extends FileContext<FileLoadingContext> {
```

## Architecture & Concepts
The ZoneFileContext is a stateful container that represents a single "Zone" definition file during the server's world generation asset loading phase. In the Hytale world structure, a Zone is a high-level geographical region that acts as a namespace and parent for a collection of Biomes.

This class serves two primary architectural purposes:

1.  **Hierarchical Grouping:** It organizes biomes into logical sets based on their source Zone file. It maintains two distinct internal registries: one for *Tile Biomes* (procedurally placed, grid-aligned biomes) and one for *Custom Biomes* (uniquely placed or special-case biomes).
2.  **Context Resolution:** It provides a mechanism to resolve and switch context based on structured file paths. The `matchContext` methods are critical for the loader, allowing it to parse a reference like "Zones.MyZone.SomeBiome" and correctly associate "SomeBiome" with the context object for "MyZone". This makes it a key routing component in the data loading pipeline.

It operates as a child of the global `FileLoadingContext`, from which it derives unique identifiers for the biomes it creates, ensuring no ID collisions occur across the entire world definition.

## Lifecycle & Ownership
-   **Creation:** A ZoneFileContext is instantiated exclusively by its parent, the `FileLoadingContext`, when a Zone asset file is discovered on the filesystem. It is never intended to be created directly.
-   **Scope:** The object's lifetime is strictly bound to the world generation asset loading process. It is created at the start of this phase and persists only until all world assets are parsed and loaded into their final runtime representations.
-   **Destruction:** It becomes eligible for garbage collection once the root `FileLoadingContext` is discarded after the loading phase completes. There are no explicit cleanup or `close` methods.

## Internal State & Concurrency
-   **State:** The ZoneFileContext is highly **mutable**. Its core purpose is to accumulate state by having `BiomeFileContext` objects registered to its internal `tileBiomes` and `customBiomes` registries during the file parsing process.
-   **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. It is designed for synchronous, single-threaded access during the asset loading pipeline. Any concurrent modification of its biome registries will result in a corrupted world state, race conditions, and unpredictable generator behavior. All interactions with an instance of this class must be confined to the main loading thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTileBiomes() | Registry | O(1) | Returns the registry for grid-based biomes. |
| getCustomBiomes() | Registry | O(1) | Returns the registry for uniquely defined biomes. |
| getBiomes(Type) | Registry | O(1) | Returns the appropriate biome registry based on the requested type. |
| matchContext(JsonElement, String) | ZoneFileContext | O(1) | Attempts to resolve a different ZoneFileContext from a JSON object. Returns self if resolution fails. |
| matchContext(String) | ZoneFileContext | O(1) | Attempts to resolve a different ZoneFileContext from a file path string. Returns self if resolution fails. |
| createBiome(String, Path, Type) | BiomeFileContext | O(1) | Factory method to create a new child BiomeFileContext, assigning it a new ID from the parent context. |

## Integration Patterns

### Standard Usage
The ZoneFileContext is manipulated by a higher-level loading coordinator. The coordinator uses it to create biomes and then registers them back into the zone's appropriate registry.

```java
// Executed within a higher-level loading system, e.g., FileLoadingContext
FileLoadingContext rootContext = ...;
ZoneFileContext currentZone = rootContext.getZones().get("EmeraldGrove");

// A new biome file is discovered that belongs to the EmeraldGrove zone
Path biomePath = Paths.get(".../biomes/ancient_forest.json");
BiomeFileContext newBiome = currentZone.createBiome("AncientForest", biomePath, BiomeFileContext.Type.Tile);

// Register the newly created biome context into the zone's registry
currentZone.getTileBiomes().register(newBiome);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ZoneFileContext()`. Doing so bypasses the parent `FileLoadingContext`'s tracking mechanisms, especially the global biome ID generator, which will lead to ID collisions and data corruption.
-   **Cross-Context Registration:** Do not create a biome using one ZoneFileContext and attempt to register it with another. The parent-child relationship is critical for context resolution and data integrity.
-   **Asynchronous Modification:** Do not access or modify a ZoneFileContext from any thread other than the one performing the initial world asset load. The internal registries are not synchronized.

## Data Pipeline
The ZoneFileContext acts as a stateful processor and container within the worldgen loading pipeline. It receives data from a parent loader and becomes the parent for subsequent biome loaders.

> Flow:
> Filesystem Scan -> `FileLoadingContext` Discovers Zone File -> **`ZoneFileContext` Instantiation** -> `FileLoadingContext` Discovers Biome File -> `createBiome` call -> `BiomeFileContext` Instantiation -> Registration into **`ZoneFileContext`**'s internal registry -> Final data passed to World Generator

