---
description: Architectural reference for PrefabContainerJsonLoader
---

# PrefabContainerJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.container
**Type:** Transient

## Definition
```java
// Signature
public class PrefabContainerJsonLoader extends JsonLoader<SeedStringResource, PrefabContainer> {
```

## Architecture & Concepts

The PrefabContainerJsonLoader is a specialized deserializer within the server's world generation asset pipeline. Its sole responsibility is to translate a specific JSON data structure into a fully realized, in-memory PrefabContainer object. This class is a critical component for data-driven world generation, allowing designers to define complex collections of structures (prefabs) and their placement rules in simple text files.

Architecturally, this class embodies the **Composite Loader Pattern**. It does not perform all the parsing logic itself. Instead, it acts as an orchestrator, delegating the parsing of nested JSON objects to other, more granular loaders. For each element in the top-level "Entries" array of the JSON, it instantiates a dedicated `PrefabContainerEntryJsonLoader`. This nested loader, in turn, uses `WeightedPrefabMapJsonLoader` and `PrefabPatternGeneratorJsonLoader` to process its own subsections.

This hierarchical approach creates a clean separation of concerns, making the world generation configuration schema extensible and the loader system easier to maintain. Each loader understands only its specific part of the JSON structure.

## Lifecycle & Ownership

-   **Creation:** An instance is created on-demand by a higher-level system within the world generation framework, likely an asset manager or a zone loader. It is provided with the raw JsonElement to parse, a file loading context, and a procedural seed. It is **not** a managed service or singleton.
-   **Scope:** Extremely short-lived. The object's lifetime is typically confined to a single method call. It is instantiated, its `load` method is called once, and then it is discarded.
-   **Destruction:** The instance becomes eligible for garbage collection immediately after the `load` method returns its PrefabContainer payload. There are no external references to the loader itself after the operation completes.

## Internal State & Concurrency

-   **State:** Immutable. The loader's internal state, consisting of the source JSON, data folder path, and loading context, is set at construction and never modified. The `load` method is a deterministic function that transforms this initial state into a PrefabContainer. It performs no caching and produces a new object on every invocation.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context as part of a sequential asset loading process. The `FileLoadingContext` it relies upon is not guaranteed to be thread-safe, and concurrent loading operations could lead to race conditions or corrupted asset data.

## API Surface

The public contract is minimal, consisting of the constructor for instantiation and the `load` method to execute the parsing operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | PrefabContainer | O(N) | Deserializes the configured JSON into a PrefabContainer object. N is the number of entries in the JSON array. Throws Error on malformed data or if a dependency (like an Environment) cannot be resolved. |

## Integration Patterns

### Standard Usage

This loader is intended to be used internally by the world generation system. A parent component responsible for loading a larger asset (like a world zone) would instantiate and use it to process a subsection of its own configuration file.

```java
// Hypothetical usage within a parent loader
FileLoadingContext loadingContext = ...;
SeedString<SeedStringResource> seed = ...;
Path dataFolder = ...;
JsonElement prefabContainerJson = zoneConfig.get("prefabs");

// The loader is created, used, and immediately discarded
PrefabContainer container = new PrefabContainerJsonLoader(seed, dataFolder, prefabContainerJson, loadingContext).load();

// The resulting container is then used by the world generator
worldGenerator.registerPrefabs(container);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Caching:** Do not retain and reuse a PrefabContainerJsonLoader instance. They are lightweight and designed to be single-use. Reusing one offers no performance benefit and can lead to unpredictable behavior if the underlying loading context changes.
-   **Manual Construction:** Avoid constructing this class manually outside of a managed asset loading pipeline. It depends on a fully populated `FileLoadingContext` to correctly resolve dependencies like Environments.
-   **Concurrent Invocation:** Never call the `load` method from multiple threads on a shared instance. This will lead to undefined behavior and likely crash the server.

## Data Pipeline

The PrefabContainerJsonLoader is a key transformation step in the data flow from disk to the game engine. It converts declarative configuration into executable game logic.

> Flow:
> `world_zone.json` -> Server Asset System -> `JsonElement` -> **PrefabContainerJsonLoader** -> `PrefabContainer` Object -> World Generator Engine

