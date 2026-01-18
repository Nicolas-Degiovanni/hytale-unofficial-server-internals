---
description: Architectural reference for ClimateGraphJsonLoader
---

# ClimateGraphJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.climate
**Type:** Transient

## Definition
```java
// Signature
public class ClimateGraphJsonLoader<K extends SeedResource> extends JsonLoader<K, ClimateGraph> {
```

## Architecture & Concepts

The ClimateGraphJsonLoader is a specialized deserializer within the server's World Generation subsystem. Its sole responsibility is to translate a declarative JSON configuration into a fully hydrated, in-memory ClimateGraph object. This class acts as a critical bridge between static worldgen asset files and the dynamic procedural generation engine.

Architecturally, this loader is a concrete implementation within a larger framework of JsonLoader classes, each tailored to a specific data structure. It embodies a hierarchical loading pattern; while it parses the top-level graph properties (like fade mode and distance), it delegates the complex task of parsing individual climate definitions to a subordinate loader, the ClimateTypeJsonLoader. This compositional approach promotes separation of concerns and allows for modular and maintainable worldgen asset definitions.

This component is a foundational piece of the data-driven world generation pipeline, enabling designers to define complex climate relationships in simple JSON files without modifying core engine code.

## Lifecycle & Ownership

-   **Creation:** An instance is created on-demand by a higher-level world generation orchestrator when it processes a worldgen configuration file containing a climate graph definition. It is never pre-initialized or managed as a persistent service.
-   **Scope:** The object's lifetime is exceptionally short. It exists only for the duration of a single `load` operation.
-   **Destruction:** The instance is immediately eligible for garbage collection once the `load` method completes and returns the resulting ClimateGraph object. It maintains no external references and is not intended to be pooled or reused.

## Internal State & Concurrency

-   **State:** The internal state of the loader consists of the seed, data folder path, and the specific JsonElement it is tasked with parsing. This state is provided at construction and is treated as immutable throughout the object's lifecycle. The `load` method is a pure function of this initial state.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution during the world generation setup phase. The underlying parent class, JsonLoader, makes no guarantees of thread safety, and concurrent access will lead to unpredictable behavior.

## API Surface

The public contract is minimal, exposing only the functionality required to perform the load operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ClimateGraph | O(N) | Deserializes the provided JSON into a ClimateGraph object. N is the number of climates defined in the JSON array. Throws exceptions if required JSON keys are missing. |

## Integration Patterns

### Standard Usage

This class is not intended for direct use by most systems. It is invoked by an orchestrating service responsible for loading all world generation assets. The typical flow involves parsing a master configuration file and delegating sections to specialized loaders like this one.

```java
// Hypothetical usage within a parent worldgen loader
JsonElement climateGraphJson = rootConfig.get("climateGraph");
SeedString<MySeed> seed = context.getWorldSeed();
Path dataFolder = context.getDataFolderPath();

// The loader is created, used once, and then discarded
ClimateGraphJsonLoader<MySeed> loader = new ClimateGraphJsonLoader<>(seed, dataFolder, climateGraphJson);
ClimateGraph graph = loader.load();

// The resulting graph is then passed to the climate simulation engine
climateSystem.registerGraph(graph);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not retain an instance of ClimateGraphJsonLoader to call `load` multiple times. Each instance is bound to a specific JsonElement at creation and is not designed for re-use.
-   **State Modification:** Do not attempt to modify the loader's internal state via reflection after construction. The loading process is predicated on the immutability of its initial configuration.
-   **Cross-thread Access:** Never pass an instance of this loader to another thread. All loading operations should occur on the main world generation thread.

## Data Pipeline

The ClimateGraphJsonLoader is a key transformation step in the world generation data pipeline. It converts raw, structured text data into a live, interconnected object graph that the engine can use for procedural calculations.

> Flow:
> WorldGen JSON File -> GSON Parser -> `JsonElement` -> **ClimateGraphJsonLoader** -> `ClimateGraph` Object -> World Generator Engine

