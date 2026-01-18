---
description: Architectural reference for MeshNoiseJsonLoader
---

# MeshNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient Loader

## Definition
```java
// Signature
public class MeshNoiseJsonLoader<K extends SeedResource> extends AbstractCellJitterJsonLoader<K, NoiseFunction> {
```

## Architecture & Concepts
The MeshNoiseJsonLoader is a specialized deserializer and factory component within the procedural generation framework. Its primary function is to translate a specific `JsonElement` configuration into a concrete, executable `NoiseFunction` object. This class embodies the engine's data-driven design philosophy, allowing world generation logic to be defined in external JSON files rather than hard-coded.

It operates as a sub-loader, typically invoked by a higher-level configuration parser when a "MeshNoise" definition is encountered. Based on the `CellType` property within the JSON—either `SQUARE` or `HEX`—it acts as a factory, instantiating either a `MeshNoise` or `HexMeshNoise` object, respectively. This isolates the parsing and validation logic for mesh-based noise patterns, such as Voronoi diagrams, into a single, focused component.

The generic parameter `K` links this loader to a broader resource tracking system, ensuring that the generated noise function is correctly seeded for deterministic and reproducible world generation.

## Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a parent JSON parsing system. It is **not a singleton or a long-lived service**. Its construction requires the specific JSON block to be processed, a `SeedString` for determinism, and a `Path` to the data folder for resolving potential sub-dependencies.
- **Scope:** The object's lifetime is exceptionally short and is scoped to a single deserialization task. It exists only to execute the `load` method once.
- **Destruction:** After the `load` method returns the `NoiseFunction`, the MeshNoiseJsonLoader instance has fulfilled its purpose and is immediately eligible for garbage collection. There is no manual cleanup or resource management required.

## Internal State & Concurrency
- **State:** The internal state, consisting of the provided JSON element, seed, and data folder path, is **effectively immutable** after construction. These fields are read during the `load` operation but are never modified. The class itself is stateless in the sense that it produces an output but does not retain any information from the operation.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for single-threaded, synchronous execution. Any parallelization of configuration loading must be managed by the calling system, which should ensure that each thread or task creates its own unique loader instances.

## API Surface
The public contract is minimal, exposing only the primary factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseFunction | O(1) | Deserializes the internal JSON data into a concrete `NoiseFunction`. Throws `IllegalStateException` if required fields like `Thickness` are missing or if the `CellType` is invalid. |

## Integration Patterns

### Standard Usage
The loader is intended to be used as a transient tool by a larger configuration processing system. The caller instantiates it for a specific JSON object, calls `load` immediately, and then discards the loader instance.

```java
// A parent configuration loader encounters a "MeshNoise" JSON object.
JsonElement meshNoiseConfig = jsonObject.get("biomeRiverNoise");
SeedString<WorldSeed> currentSeed = getSeedFor("biomeRiverNoise");
Path dataPath = Paths.get("./data");

// It instantiates the loader for this specific sub-task.
MeshNoiseJsonLoader<WorldSeed> loader = new MeshNoiseJsonLoader<>(currentSeed, dataPath, meshNoiseConfig);

// The loader is used immediately to produce the final object.
NoiseFunction resultingNoise = loader.load();

// The resultingNoise object is now integrated into the world generator.
// The loader instance is no longer needed and will be garbage collected.
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache or reuse instances of MeshNoiseJsonLoader. Each instance is tied to a specific `JsonElement` and is not designed for repeated use.
- **State Modification:** Do not attempt to modify the loader's internal state via reflection. The object's integrity depends on its immutability post-construction.
- **Ignoring Exceptions:** The `load` method performs critical validation. Failure to catch potential `IllegalStateException` or other runtime exceptions will result in unhandled configuration errors and terminate the generation process.

## Data Pipeline
The MeshNoiseJsonLoader is a single, focused step in a larger data transformation pipeline that converts declarative configuration into executable game logic.

> Flow:
> JSON File on Disk -> Engine Configuration Service -> `JsonElement` (for MeshNoise) -> **MeshNoiseJsonLoader** -> `NoiseFunction` Instance -> Procedural Generation Engine -> World Data

