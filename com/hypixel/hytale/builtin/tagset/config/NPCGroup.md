---
description: Architectural reference for NPCGroup
---

# NPCGroup

**Package:** com.hypixel.hytale.builtin.tagset.config
**Type:** Asset / Data Model

## Definition
```java
// Signature
public class NPCGroup implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, NPCGroup>>, TagSet {
```

## Architecture & Concepts

The NPCGroup class is a data-driven configuration asset, not a service or manager. Its primary architectural role is to define a logical collection of NPC types using a declarative, set-based model of inclusion and exclusion. This allows designers and modders to create complex NPC groupings—such as *all monsters except zombies* or *all friendly animals and golems*—without writing code.

It is a fundamental building block of the Hytale **TagSet** system, a generic framework for applying and resolving tags. An NPCGroup is loaded from a corresponding JSON file by the engine's core **AssetStore** framework. Game systems do not instantiate this class directly; they retrieve fully-formed, read-only instances from the central asset repository.

The static **CODEC** field is the cornerstone of this class's design. It is an AssetBuilderCodec that defines the public JSON contract for this asset type. It dictates how raw data from a file is mapped to the fields of an NPCGroup object, effectively serving as both the deserializer and the schema definition.

## Lifecycle & Ownership

-   **Creation:** NPCGroup instances are created exclusively by the Hytale AssetStore during the engine's asset loading phase. The static CODEC field is used to deserialize a JSON definition from disk into a hydrated Java object. Direct instantiation via `new NPCGroup()` is strictly forbidden and will result in a non-functional object.

-   **Scope:** Application-scoped. Once an NPCGroup asset is loaded, it is cached in a static AssetStore for the entire duration of the game session. It is treated as immutable configuration data.

-   **Destruction:** Instances are destroyed when the AssetRegistry is shut down or reloaded, typically on game exit. The static ASSET_STORE field holds the strong reference, so memory is reclaimed only when its classloader is unloaded or the store is explicitly cleared.

## Internal State & Concurrency

-   **State:** The internal state of an NPCGroup instance—comprising its ID and its four inclusion/exclusion arrays—is **effectively immutable** post-deserialization. There are no public methods to alter its state after it has been loaded. This design ensures predictable and safe read-only access throughout the application.

-   **Thread Safety:** The class is **thread-safe for all read operations**. Because its state is fixed at load time, multiple systems (e.g., AI, Spawning, Faction Logic) can safely query NPCGroup instances from any thread without requiring locks or synchronization.

    **Warning:** The lazy initialization within the static `getAssetStore` method presents a potential race condition if accessed by multiple threads before the central AssetRegistry is fully initialized. The engine's bootstrap sequence mitigates this by ensuring all asset stores are prepared on the main thread before worker threads are dispatched.

## API Surface

The public API is designed for querying the asset repository and reading the resolved tag set data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the global, shared repository for all NPCGroup assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Returns the high-performance lookup map from the asset store. |
| getId() | String | O(1) | Returns the unique asset key for this group. |
| getIncludedTagSets() | String[] | O(1) | Returns asset keys of other NPCGroups to include in this set. |
| getExcludedTagSets() | String[] | O(1) | Returns asset keys of other NPCGroups to exclude from this set. |
| getIncludedTags() | String[] | O(1) | Returns asset keys of individual NPC roles to include. |
| getExcludedTags() | String[] | O(1) | Returns asset keys of individual NPC roles to exclude. |

## Integration Patterns

### Standard Usage

Game systems should always access NPCGroup definitions through the static `getAssetStore` or `getAssetMap` methods. This ensures access to the canonical, engine-managed instance.

```java
// Retrieve the central repository for all NPCGroup assets
IndexedLookupTableAssetMap<String, NPCGroup> assetMap = NPCGroup.getAssetMap();

// Look up a specific group by its asset key (e.g., "hytale:forest_predators")
NPCGroup predatorGroup = assetMap.get("hytale:forest_predators");

if (predatorGroup != null) {
    // The TagSetResolver system would typically consume this object
    // to determine the final, flattened list of NPC types.
    String[] includedNpcs = predatorGroup.getIncludedTags();
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new NPCGroup()`. The resulting object will be an empty shell, unmanaged by the engine and lacking any configuration data. It will not be registered in the asset store and will be invisible to other game systems.

-   **State Mutation:** Do not attempt to modify the arrays returned by the getter methods. While the language does not prevent this, the asset system assumes this data is read-only. Modifying it would lead to unpredictable behavior and desynchronization across the engine.

-   **Early Access:** Do not call `getAssetStore()` from a background thread during the earliest phases of engine startup. Always ensure the main thread has completed the AssetRegistry initialization first.

## Data Pipeline

The NPCGroup class is a destination for data loaded from game files. It does not process a continuous stream of data at runtime. Its pipeline is one of deserialization and registration.

> Flow:
> JSON File on Disk (e.g., `monsters.json`) -> Engine Asset Loader -> **NPCGroup.CODEC** (Deserialization) -> In-Memory **NPCGroup** Instance -> Registration in static **AssetStore** -> Read-only access by Game Systems (e.g., World Spawner)

