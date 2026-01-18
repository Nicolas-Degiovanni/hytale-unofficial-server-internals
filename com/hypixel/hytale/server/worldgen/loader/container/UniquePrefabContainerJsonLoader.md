---
description: Architectural reference for UniquePrefabContainerJsonLoader
---

# UniquePrefabContainerJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.container
**Type:** Transient

## Definition
```java
// Signature
public class UniquePrefabContainerJsonLoader extends JsonLoader<SeedStringResource, UniquePrefabContainer> {
```

## Architecture & Concepts

The UniquePrefabContainerJsonLoader is a specialized factory component within the server-side World Generation pipeline. Its sole responsibility is to deserialize a specific JSON structure from a zone file and construct a fully-hydrated UniquePrefabContainer runtime object.

This class operates as a node in a hierarchical, compositional loading system. It is not a standalone service but is rather invoked by a higher-level loader responsible for parsing an entire zone. It, in turn, delegates the loading of more granular components—specifically individual UniquePrefabGenerator configurations—to its inner class, UniquePrefabGeneratorJsonLoader.

A critical architectural feature is its dependency on the ZoneFileContext. This context object acts as a dependency injection mechanism, providing the loader with essential information from the broader world generation environment, such as the set of valid PrefabCategory definitions. This design prevents the loader from operating in isolation and ensures that the generated objects are consistent with the overall world configuration.

## Lifecycle & Ownership

-   **Creation:** An instance is created on-demand by a parent loader (e.g., a ZoneLoader) when it encounters the JSON key corresponding to unique prefab definitions. It is not a singleton or a managed service.
-   **Scope:** The object's lifetime is exceptionally short and is strictly confined to the scope of a single `load` method execution.
-   **Destruction:** Once the `load` method returns the constructed UniquePrefabContainer, the loader instance holds no further purpose. It becomes unreferenced and is immediately eligible for garbage collection. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency

-   **State:** The internal state of the loader (seed, dataFolder, json, zoneContext) is provided at construction and is treated as **immutable** for the object's lifetime. The class is stateless in the sense that it performs a pure transformation from input JSON to an output object without retaining any information between calls.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. The world generation loading process is designed to be a deterministic, single-threaded sequence. Sharing instances of this loader or its context across threads will result in unpredictable behavior and potential race conditions.

## API Surface

The public contract is minimal, centered entirely on the `load` method. The constructor is public for framework use but is not part of the intended developer-facing API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | UniquePrefabContainer | O(N) | Deserializes the configured JSON into a container object. N is the number of entries in the "Entries" JSON array. Throws JsonParseException on malformed data. |

## Integration Patterns

### Standard Usage

This class is an internal framework component and is not intended for direct use by game or mod developers. It is invoked automatically by the world generation system during zone loading. The following example is a hypothetical illustration of how the framework uses it.

```java
// This class is invoked by a higher-level loader, such as a ZoneLoader.
// The following is a conceptual example and not for direct implementation.

JsonElement uniquePrefabsJson = zoneDefinition.get("uniquePrefabs");
ZoneFileContext context = getZoneFileContext();
SeedString currentSeed = getScopedSeed();
Path dataPath = getDataFolderPath();

// Framework instantiates the loader with necessary context
UniquePrefabContainerJsonLoader loader = new UniquePrefabContainerJsonLoader(
    currentSeed,
    dataPath,
    uniquePrefabsJson,
    context
);

// The loaded container is passed to the world generator
UniquePrefabContainer container = loader.load();
zoneGenerator.setUniquePrefabs(container);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never manually construct this class with `new`. The world generation framework is responsible for providing the correct SeedString and ZoneFileContext. Manual creation will bypass critical contextual validation and lead to invalid or corrupt world data.
-   **Instance Caching:** Do not cache or reuse instances of this loader. It is designed as a single-use, transient object. Each JSON block must be processed by a new loader instance.
-   **Context Manipulation:** Do not attempt to modify the ZoneFileContext after passing it to the loader. The loading process assumes the context is stable for the duration of the `load` call.

## Data Pipeline

The UniquePrefabContainerJsonLoader functions as a specific step in the data transformation pipeline that converts static asset files into live game objects.

> Flow:
> Zone Definition File (JSON) -> Server WorldGen Engine -> **UniquePrefabContainerJsonLoader** -> `UniquePrefabContainer` (In-Memory Object) -> ZoneGenerator Engine

