---
description: Architectural reference for BalanceAsset
---

# BalanceAsset

**Package:** com.hypixel.hytale.server.npc.config.balancing
**Type:** Data Model / Asset

## Definition
```java
// Signature
public class BalanceAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, BalanceAsset>> {
```

## Architecture & Concepts

The BalanceAsset class is a passive data model that represents a specific set of combat balancing parameters for an NPC. It is a cornerstone of Hytale's data-driven architecture, allowing designers to define and tune NPC behavior in external JSON files without modifying engine code.

This class is not a service or a manager; it is a schema for configuration. Its primary role is to be instantiated by the asset loading system from a corresponding JSON definition. The static fields, particularly the various `Codec` implementations (`ABSTRACT_CODEC`, `BASE_CODEC`, `CODEC`), are the most critical components. They form a self-describing contract that dictates how JSON data is deserialized into a Java BalanceAsset object.

Architecturally, BalanceAsset integrates deeply with the global `AssetRegistry` and `AssetStore` systems. The class itself acts as a typed key to retrieve its corresponding `AssetStore`, which serves as a central, in-memory database for all loaded balancing configurations. The `entityEffect` field demonstrates a common pattern in this system: a string-based foreign key that references another asset type, `EntityEffect`, creating a relational link between different configuration assets.

## Lifecycle & Ownership

-   **Creation:** Instances are **never** created directly using the `new` keyword. They are instantiated exclusively by the Hytale `AssetStore` during the server's asset loading phase. The `BalanceAsset.CODEC` is invoked to deserialize a JSON file from the server's assets, populating a new BalanceAsset object.
-   **Scope:** An instance of BalanceAsset, once loaded, is a shared, effectively immutable singleton for its given ID. It persists in the static `ASSET_STORE` for the entire lifetime of the server session.
-   **Destruction:** All instances are destroyed when the `AssetStore` is cleared, which typically occurs during a full server shutdown or a manual asset reload command. Individual objects are not garbage collected until the entire store is dereferenced.

## Internal State & Concurrency

-   **State:** The state of a BalanceAsset instance is **immutable** after its initial creation and loading by the asset system. The fields `id` and `entityEffect` are populated once during deserialization and are not designed to be changed at runtime.

-   **Thread Safety:** The class is **conditionally thread-safe**.
    -   **Instance Safety:** Individual BalanceAsset objects are safe to read from multiple threads due to their immutable nature post-initialization.
    -   **Static Safety:** The static `getAssetStore` method features a lazy initialization check. While this could theoretically present a race condition, the entire asset system is initialized within a single-threaded bootstrap context at server startup, mitigating this risk in practice.
    -   **WARNING:** Accessing the static `ASSET_STORE` before the main asset loading phase is complete will result in a null reference or an incomplete data set.

## API Surface

The public API is primarily for data retrieval from the central store.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the global, shared AssetStore for all BalanceAsset configurations. |
| getAssetMap() | static DefaultAssetMap | O(1) | Convenience method to retrieve the map of all loaded BalanceAsset instances, keyed by ID. |
| getId() | String | O(1) | Returns the unique identifier for this specific balance configuration. |
| getEntityEffect() | String | O(1) | Returns the asset key for the EntityEffect to be applied. |

## Integration Patterns

### Standard Usage

The standard pattern is to retrieve a specific, pre-defined configuration from the global asset map using its known ID. This is typically done by systems that need to configure a new entity, such as an NPC spawner.

```java
// Retrieve the map of all available balance configurations
DefaultAssetMap<String, BalanceAsset> allBalanceConfigs = BalanceAsset.getAssetMap();

// Fetch a specific configuration by its string ID, defined in a JSON file
BalanceAsset config = allBalanceConfigs.get("zombie_brute_tier_2");

if (config != null) {
    // Use the data to configure an NPC
    String effectKey = config.getEntityEffect();
    // ... logic to find and apply the EntityEffect asset
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BalanceAsset()`. The constructor is protected and intended for use only by the asset deserialization system. Bypassing the `AssetStore` breaks the single source of truth principle.
-   **Runtime Modification:** Do not attempt to modify a BalanceAsset instance after retrieving it. These objects are shared across the entire server; any modification would create unpredictable side effects. Treat them as read-only data records.
-   **Local Caching:** Do not cache the result of `getAssetMap().get("some_id")` in long-lived objects. While efficient, this can cause issues during asset hot-reloading, where your local reference may become stale. It is safer to re-fetch from the map when needed.

## Data Pipeline

The flow of data from a designer's configuration file to its use in the game engine is a one-way pipeline.

> Flow:
> JSON File on Disk -> Server Asset Loader -> **BalanceAsset.CODEC** (Deserialization) -> In-Memory `AssetStore` -> Game System (e.g., NPC Spawner) retrieves **BalanceAsset** instance -> NPC Configuration<ctrl63>

