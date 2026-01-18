---
description: Architectural reference for BarterShopAsset
---

# BarterShopAsset

**Package:** com.hypixel.hytale.builtin.adventure.shop.barter
**Type:** Data Asset

## Definition
```java
// Signature
public class BarterShopAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, BarterShopAsset>> {
```

## Architecture & Concepts

The BarterShopAsset class is not a service or a manager, but a data-defined asset that represents the static configuration of a single barter shop within the game world. It serves as the in-memory representation of a data file, likely a JSON file, that specifies a shop's properties, such as its available trades, refresh intervals, and display name.

This class is a fundamental component of the Hytale **Asset System**. Its primary architectural feature is the static **CODEC** field, an AssetBuilderCodec instance. This codec declaratively defines the mapping between the source data format (JSON) and the Java object's fields. This pattern decouples the game's data authoring from its runtime representation, allowing designers to configure game systems without modifying engine code.

Instances of BarterShopAsset are not managed by game logic directly. Instead, they are loaded, cached, and managed by the global **AssetStore**, which acts as a centralized repository. The class provides a static `getAssetStore` method, a common service locator pattern within the engine, to access this central registry. This ensures that there is only one instance of each unique barter shop asset in memory, which is then shared read-only across the entire application.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale AssetStore during the engine's asset loading phase. The `CODEC` is invoked by the store to deserialize a source file (e.g., `goblin_trader.json`) into a new BarterShopAsset object. The protected no-argument constructor exists solely for this deserialization process.
-   **Scope:** An instance of BarterShopAsset lives for the entire duration that its corresponding asset pack is loaded. Typically, this means it is loaded at game startup and persists in the AssetStore until the client is shut down or a hot-reload of assets is triggered.
-   **Destruction:** The lifecycle is managed entirely by the AssetStore. When the store is cleared or reloaded, it drops its references to the asset objects, making them eligible for garbage collection. There is no manual destruction or cleanup method.

## Internal State & Concurrency

-   **State:** The state of a BarterShopAsset is considered **immutable after initialization**. While its fields are not marked as final, they are populated only once during the deserialization process by the CODEC. After being published to the AssetStore, the object should be treated as a read-only data container.

-   **Thread Safety:** The object is **thread-safe for read operations**. Because its internal state does not change after being loaded, multiple threads (e.g., game logic, rendering, UI) can safely call its getter methods without external synchronization.

    **WARNING:** The static `getAssetStore` method uses lazy initialization. Calling this method from multiple threads simultaneously before the store is initialized is not a supported operation and may lead to race conditions. Asset stores are expected to be initialized from the main thread during the game's bootstrap sequence.

## API Surface

The public API is designed for read-only access to the asset's configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, shared repository for all BarterShopAsset instances. |
| getAssetMap() | static DefaultAssetMap | O(1) | Convenience method to get the asset map directly from the global store. |
| getId() | String | O(1) | Returns the unique identifier for this shop asset, e.g., "adventure:goblin_trader". |
| getDisplayNameKey() | String | O(1) | Returns the localization key for the shop's user-facing name. |
| getRefreshInterval() | RefreshInterval | O(1) | Returns the data structure defining when the shop's inventory refreshes. |
| getTrades() | BarterTrade[] | O(1) | Returns the array of trades available at this shop. |
| getRestockHour() | int | O(1) | Returns the in-game hour (0-23) when stock is replenished. Defaults to 7. |

## Integration Patterns

### Standard Usage

The correct way to use this asset is to retrieve it from the central AssetMap using its unique string identifier. Never create an instance directly.

```java
// Retrieve the central map of all loaded barter shop assets
DefaultAssetMap<String, BarterShopAsset> allShops = BarterShopAsset.getAssetMap();

// Fetch a specific shop's configuration using its known ID
BarterShopAsset goblinShop = allShops.get("adventure:goblin_trader");

if (goblinShop != null) {
    // Use the read-only data to configure a live game entity or UI
    BarterTrade[] trades = goblinShop.getTrades();
    System.out.println("Shop Name Key: " + goblinShop.getDisplayNameKey());
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BarterShopAsset()`. This creates an unmanaged, un-cached object that is disconnected from the engine's asset system. It will not be accessible via `getAssetMap` and circumvents the entire asset pipeline.

-   **State Mutation:** The arrays returned by methods like `getTrades` are direct references to the internal state of the shared asset. Modifying the contents of this array will affect all systems that use this asset, leading to unpredictable behavior and data corruption. Treat all returned data as strictly read-only.

## Data Pipeline

The BarterShopAsset is the result of a data transformation pipeline managed by the engine's asset loading system.

> Flow:
> `*.json` file on disk -> AssetStore Loader -> **BarterShopAsset.CODEC** (Deserialization) -> **BarterShopAsset Instance** -> Caching in AssetStore -> Read-only access by Game Systems<ctrl63>

