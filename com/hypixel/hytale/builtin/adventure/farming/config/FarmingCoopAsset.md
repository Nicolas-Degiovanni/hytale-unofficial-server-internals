---
description: Architectural reference for FarmingCoopAsset
---

# FarmingCoopAsset

**Package:** com.hypixel.hytale.builtin.adventure.farming.config
**Type:** Data Transfer Object / Asset

## Definition
```java
// Signature
public class FarmingCoopAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, FarmingCoopAsset>> {
```

## Architecture & Concepts

The FarmingCoopAsset class is a data-driven configuration object that defines the behavior and properties of a farming coop structure within the game world. It serves as a blueprint, deserialized from JSON files at game startup, allowing designers to configure coop mechanics without modifying engine code.

This class is a central component of the Hytale **Asset System**. Its primary architectural feature is the static `CODEC` field, an instance of AssetBuilderCodec. This codec dictates the entire serialization and deserialization contract between the JSON configuration files and the in-memory Java object. It maps JSON keys like "MaxResidents" and "ProduceDrops" directly to the fields of the class.

A key architectural pattern demonstrated here is the **post-deserialization processing** via the `afterDecode` hook. This hook transforms the human-readable `acceptedNpcGroupIds` (an array of strings) into `acceptedNpcGroupIndexes` (an array of integers). This is a critical runtime optimization that converts string-based lookups into much faster integer-based index lookups for use in performance-sensitive game logic.

This asset acts as read-only data for higher-level game systems, such as an NPC management or farming simulation system, which query the global AssetStore to retrieve these configurations at runtime.

### Lifecycle & Ownership
- **Creation:** Instances are not created manually via the `new` keyword. They are instantiated exclusively by the Hytale AssetStore during the engine's asset loading phase. The static `CODEC` is used to parse corresponding JSON files and construct the objects.
- **Scope:** An instance of FarmingCoopAsset, once loaded, persists for the entire game session (client or server). All loaded assets are held in a static, globally accessible `AssetStore`.
- **Destruction:** The objects are garbage collected by the JVM when the game process terminates and the AssetRegistry is cleared. There is no manual deallocation.

## Internal State & Concurrency
- **State:** The state of a FarmingCoopAsset is considered **effectively immutable** after the `afterDecode` process completes. While its fields are not marked as `final`, the governing architectural pattern is that these assets are loaded once and then only read from. Modifying a loaded asset at runtime is a severe anti-pattern that would result in non-deterministic behavior.
- **Thread Safety:** The object is safe for concurrent reads from multiple threads *after* the initial asset loading phase is complete. However, the lazy initialization within the `getAssetStore` method is **not thread-safe**.

    **WARNING:** The `if (ASSET_STORE == null)` check is unsynchronized. If multiple threads call this method simultaneously before the store is initialized, it could result in multiple AssetStore instances being created or other race conditions. The engine's design assumes that all asset stores are initialized from a single thread during game bootstrap.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | AssetBuilderCodec | N/A | **STATIC**. The core deserializer that defines the mapping from JSON to this class. Used internally by the asset system. |
| getAssetStore() | AssetStore | O(1) | **STATIC**. Retrieves the singleton store containing all loaded FarmingCoopAsset instances. Not thread-safe on first call. |
| getAssetMap() | DefaultAssetMap | O(1) | **STATIC**. Convenience method to get the underlying map of all assets, keyed by their string ID. |
| getAcceptedNpcGroupIndexes() | int[] | O(1) | Returns the cached, integer-based indices for accepted NPC groups. This is the optimized version for runtime checks. |
| getMaxResidents() | int | O(1) | Returns the maximum number of NPCs that can reside in the coop. |
| getProduceDrops() | Map | O(1) | Returns the mapping of conditions to item drop lists for produce generation. |

## Integration Patterns

### Standard Usage
Game logic should never instantiate this class directly. Instead, it should retrieve the global asset map and query for a specific configuration by its unique asset identifier.

```java
// Retrieve the central map of all coop configurations
DefaultAssetMap<String, FarmingCoopAsset> coopAssets = FarmingCoopAsset.getAssetMap();

// Get the specific configuration for a "standard_chicken_coop"
FarmingCoopAsset chickenCoopConfig = coopAssets.get("hytale:standard_chicken_coop");

if (chickenCoopConfig != null) {
    int maxResidents = chickenCoopConfig.getMaxResidents();
    // ... use configuration to drive game logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new FarmingCoopAsset()`. The resulting object will be uninitialized, lack critical data like its ID, and will not be registered in the global `AssetStore`. All game systems rely on the centrally managed instances.
- **Runtime Modification:** Do not modify the state of a FarmingCoopAsset instance after retrieving it from the `AssetStore`. These objects are shared, global configurations. Modifying one will affect all systems that use it, leading to unpredictable and hard-to-debug issues.
- **Concurrent Initialization:** Do not call `getAssetStore()` or `getAssetMap()` from multiple threads during the application's initial loading phase. This can break the singleton pattern of the asset store.

## Data Pipeline
The flow of data from configuration file to usable game object follows a clear, multi-stage pipeline managed by the engine's asset system.

> Flow:
> JSON File on Disk -> Engine AssetLoader -> **FarmingCoopAsset.CODEC** (Deserialization) -> `afterDecode` Hook (Data Transformation & Optimization) -> Instance stored in **AssetStore** -> Game Logic requests instance by ID

