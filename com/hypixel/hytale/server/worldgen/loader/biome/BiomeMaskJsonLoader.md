---
description: Architectural reference for BiomeMaskJsonLoader
---

# BiomeMaskJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.biome
**Type:** Transient

## Definition
```java
// Signature
public class BiomeMaskJsonLoader extends JsonLoader<SeedStringResource, IIntCondition> {
```

## Architecture & Concepts
The BiomeMaskJsonLoader is a specialized parser within the world generation data loading pipeline. Its primary function is to translate a human-readable list of biome selection rules from a JSON source into a machine-optimized predicate object, an IIntCondition. This predicate is used by the procedural generation engine to rapidly determine if a given biome ID is part of a specified group.

This class acts as a critical bridge between declarative world configuration (JSON files) and the imperative logic of the world generator. It decodes a simple but powerful rule syntax that allows designers to select biomes based on their name, their type (e.g., TILE, CUSTOM), and the zone they belong to.

The rule syntax follows the pattern: `[zoneName.]biomeName[#biomeType]`.
-   `plains`: Selects the biome named "plains" in the current zone context.
-   `zone1.forest`: Selects the biome named "forest" from the zone named "zone1".
-   `*#TILE`: Selects all biomes of type TILE in the current zone.
-   `zone2.*`: Selects all biomes of any type from the zone named "zone2".

The loader resolves these string-based rules into a collection of integer biome IDs, which are then compiled into a final IIntCondition for high-performance evaluation during world generation.

## Lifecycle & Ownership
-   **Creation:** An instance is created by a higher-level configuration loader (e.g., a Zone loader) whenever it encounters a JSON property that defines a biome mask. It is instantiated with all necessary context, including the raw JSON data, the current zone context, and a world generation seed.

-   **Scope:** The BiomeMaskJsonLoader object is **ephemeral and single-use**. Its lifespan is strictly limited to the duration of a single `load` method call. It is a transient utility, not a persistent service.

-   **Destruction:** The object is eligible for garbage collection immediately after the `load` method returns. The result of its operation, the IIntCondition, is what persists. This result is typically stored in a long-lived cache, the `FileMaskCache`, managed by the SeedStringResource.

## Internal State & Concurrency
-   **State:** The class maintains mutable internal state, including `zoneContext` and `fileName`. The `fileName` field is notably set as a side-effect during the file loading process inherited from its parent, `JsonLoader`. This state is specific to a single, isolated loading operation.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated, used, and discarded within the scope of a single thread.

    **Warning:** Do not share instances of BiomeMaskJsonLoader across multiple threads. Concurrent calls to `load` on the same instance will lead to race conditions and unpredictable behavior, particularly regarding the resolution of the internal `fileName` state. The underlying caching mechanisms it interacts with (e.g., `FileMaskCache`) are assumed to be thread-safe, but the loader instance itself is not.

## API Surface
The public API is minimal, exposing only the primary loading operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | IIntCondition | O(N) | Parses the JSON source and returns a biome mask predicate. N is the number of rules. Complexity can increase if wildcard rules require iterating over large biome registries. This method is idempotent and heavily cached. |

## Integration Patterns

### Standard Usage
The loader is intended to be used in a "fire-and-forget" manner. A parent system instantiates it with the required context, invokes `load`, and immediately discards the loader instance, retaining only the resulting IIntCondition.

```java
// Context provided by a higher-level world generation loader
ZoneFileContext currentZone = ...;
SeedString<SeedStringResource> seed = ...;
JsonElement maskJson = ...; // The JSON array of rules

// Instantiate, use, and discard
BiomeMaskJsonLoader loader = new BiomeMaskJsonLoader(seed, dataFolderPath, maskJson, "surfaceBiomes", currentZone);
IIntCondition surfaceBiomeMask = loader.load();

// The 'surfaceBiomeMask' is now used by the generator. The 'loader' instance is no longer needed.
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not retain and re-use a BiomeMaskJsonLoader instance to load a different mask. Each instance is configured for a single, specific mask definition.

-   **State Manipulation:** Do not attempt to modify the internal state of the loader after construction. Its behavior is dependent on the context provided at creation time.

-   **Concurrent Access:** As detailed under Concurrency, never call `load` from multiple threads on the same loader instance.

## Data Pipeline
The loader transforms declarative string rules into a performant, integer-based data structure. The flow of data is unidirectional and involves multiple layers of caching.

> Flow:
> JSON File on Disk -> `FileMaskCache` (File Content Cache) -> `JsonElement` -> **BiomeMaskJsonLoader** -> Rule Parsing & Biome ID Resolution -> `IntConditionBuilder` -> `IIntCondition` -> `FileMaskCache` (Parsed Object Cache)

