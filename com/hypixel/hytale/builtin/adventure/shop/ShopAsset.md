---
description: Architectural reference for ShopAsset
---

# ShopAsset

**Package:** com.hypixel.hytale.builtin.adventure.shop
**Type:** Data Asset

## Definition
```java
// Signature
public class ShopAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, ShopAsset>> {
```

## Architecture & Concepts

The ShopAsset class is a data-driven component that represents the structure and content of an in-game shop. It is not a service or manager; rather, it is a passive, in-memory representation of data defined in external JSON files. This class serves as the critical link between on-disk configuration and the runtime game logic that presents shop interfaces to players.

Its primary architectural role is to integrate seamlessly with the engine's core **Asset System**. This integration is achieved via several static members:

*   **CODEC:** An AssetBuilderCodec that defines the contract for deserializing a JSON object into a hydrated ShopAsset instance. It maps JSON keys, such as "Content", to the corresponding Java fields.
*   **ASSET_STORE:** A static, lazily-initialized reference to the global repository for all loaded ShopAsset instances. This store acts as a session-wide cache, ensuring that each unique shop is loaded from disk only once.
*   **VALIDATOR_CACHE:** Provides a mechanism for other systems to validate that a given string ID corresponds to a valid, loaded ShopAsset without needing to retrieve the full object.

Game systems do not interact with raw shop JSON files. Instead, they request a fully parsed and validated ShopAsset object from the central AssetStore using a unique string identifier.

## Lifecycle & Ownership

The lifecycle of a ShopAsset instance is strictly managed by the Hytale Asset System, not by game logic developers.

*   **Creation:** Instances are created exclusively by the AssetStore during the engine's asset loading phase at startup or during a hot-reload. The static CODEC field is invoked by the store to parse a corresponding JSON file and construct the object. Manual instantiation is an anti-pattern and will result in an unmanaged object.

*   **Scope:** An instance of ShopAsset, once loaded, persists for the entire server or client session. It is stored and owned by the static ASSET_STORE, which acts as a singleton cache for this asset type. All requests for an asset with the same ID will receive a reference to the same, single instance.

*   **Destruction:** Objects are eligible for garbage collection only when the AssetStore is cleared. This typically occurs during a full asset reload or when the server or client shuts down.

## Internal State & Concurrency

*   **State:** A ShopAsset object is **effectively immutable** after its creation by the AssetStore. Its fields, including the id and the array of ChoiceElements, are populated once during deserialization and are not designed to be modified at runtime. It should be treated as a read-only data container.

*   **Thread Safety:** The class is **thread-safe for read operations**. Because its internal state does not change after initialization, multiple game threads can safely access its methods (e.g., getElements) without external locking.

    **Warning:** The lazy initialization within the static getAssetStore method relies on the broader engine assumption that the AssetRegistry is fully populated on the main thread before any multi-threaded game logic begins. Calling this method from multiple threads before its first successful execution is not guaranteed to be safe.

## API Surface

The public API is minimal, focusing on data access and interaction with the central asset system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) amortized | Retrieves the singleton store for all ShopAsset instances. |
| getAssetMap() | static DefaultAssetMap | O(1) | Returns a direct map of all loaded ShopAssets, keyed by ID. |
| getId() | String | O(1) | Returns the unique identifier for this shop asset. |
| getElements() | ChoiceElement[] | O(1) | Returns the array of choices that constitute the shop's content. |

## Integration Patterns

### Standard Usage

A ShopAsset should never be instantiated directly. It should always be retrieved from the central AssetStore or via a system that resolves asset keys.

```java
// Correct: Retrieve a pre-loaded ShopAsset from the central store
// In a real system, you would likely use a service that wraps this logic.

// Get the map of all loaded shops
DefaultAssetMap<String, ShopAsset> allShops = ShopAsset.getAssetMap();

// Retrieve a specific shop by its unique ID
ShopAsset townSquareShop = allShops.get("adventure:shops/town_square_general_store");

if (townSquareShop != null) {
    // Use the asset data to populate a UI or process a transaction
    ChoiceElement[] items = townSquareShop.getElements();
    // ...
}
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Do not use `new ShopAsset()`. This bypasses the asset management system, creating a "phantom" object that is not cached, not validated, and will not be recognized by other engine systems.

    ```java
    // INCORRECT
    ShopAsset myShop = new ShopAsset("my.custom.id", new ChoiceElement[0]);
    // This object is not registered in the AssetStore.
    ```

*   **State Mutation:** Do not modify the contents of the array returned by getElements. A ShopAsset is a shared, global resource. Modifying it can lead to unpredictable behavior and state corruption across the entire application.

    ```java
    // INCORRECT
    ShopAsset shop = ShopAsset.getAssetMap().get("some_shop");
    shop.getElements()[0] = null; // DANGEROUS: This mutates shared global state.
    ```

## Data Pipeline

The ShopAsset transforms a static data file into a usable runtime object through a well-defined pipeline managed by the engine.

> Flow:
> JSON File on Disk (`/assets/.../shops/my_shop.json`) -> Engine Asset Loader -> **ShopAsset.CODEC** (Deserialization) -> Instantiated **ShopAsset** Object -> Cached in **ShopAsset.ASSET_STORE** -> Game Logic (Request by ID) -> UI / Gameplay System<ctrl63>

