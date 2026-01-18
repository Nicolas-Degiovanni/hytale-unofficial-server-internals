---
description: Architectural reference for ItemQuality
---

# ItemQuality

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Data Asset

## Definition
```java
// Signature
public class ItemQuality
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, ItemQuality>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ItemQuality> {
```

## Architecture & Concepts

The ItemQuality class is a data-centric asset model that defines the properties of an item's quality tier, such as *Common*, *Rare*, or *Legendary*. It is not a service or manager, but rather a passive data container that represents configuration loaded from JSON files at server startup.

This class serves as a critical link between game data configuration and runtime behavior. It dictates visual aspects like tooltip textures, slot borders, and text colors, as well as gameplay-relevant properties like visibility in creative menus.

Its primary architectural role is to be a deserialization target for the **AssetStore** system. The static **CODEC** field, an **AssetBuilderCodec**, declaratively defines the object's schema, validation rules, and mapping from JSON keys to Java fields. This pattern centralizes the definition of the asset, ensuring that all loaded ItemQuality instances are valid and consistently structured. The class also implements **NetworkSerializable**, indicating its data can be packaged into a lightweight Data Transfer Object (DTO) and sent to game clients.

## Lifecycle & Ownership

-   **Creation:** ItemQuality instances are not created directly by developers. They are instantiated and populated by the **AssetRegistry** during the server's bootstrap phase. The framework reads all corresponding JSON asset files, using the static **CODEC** to construct each object. A special static instance, **DEFAULT_ITEM_QUALITY**, is created at class-load time to serve as a fallback.
-   **Scope:** An instance of ItemQuality, once loaded, is a global singleton for its unique asset ID (e.g., "Default", "Rare"). It persists for the entire lifetime of the server application.
-   **Destruction:** Instances are destroyed only when the server shuts down and the Java Virtual Machine garbage collects the entire **AssetRegistry**.

## Internal State & Concurrency

-   **State:** The state of an ItemQuality object is intended to be **immutable** after its initial creation by the asset loading system. It holds configuration data. The one exception is the transient `cachedPacket` field, which uses a **SoftReference** for performance-optimized caching of the network packet representation. This cache is mutable but its management is self-contained.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. However, it is safe for concurrent use under the intended operational paradigm: write-once during the single-threaded server startup, and read-many from multiple game threads thereafter.

    **WARNING:** Any modification to an ItemQuality object's fields after the initial asset load will introduce race conditions and lead to unpredictable, inconsistent behavior across the server. This is a severe violation of the asset system's contract.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | Static accessor for the global registry of all ItemQuality assets. |
| getAssetMap() | IndexedLookupTableAssetMap | O(1) | Returns the underlying map of all loaded ItemQuality assets from the store. |
| toPacket() | com.hypixel.hytale.protocol.ItemQuality | O(1) | Converts the asset into its network DTO. Uses a SoftReference cache. |
| getId() | String | O(1) | Returns the unique identifier for this quality (e.g., "Rare"). |
| get...() | various | O(1) | Provides read-only access to configuration properties. |

## Integration Patterns

### Standard Usage

The correct pattern is to retrieve a managed instance from the global **AssetStore** via the static accessor. Never store a local copy; always request it from the registry to ensure you have the canonical instance.

```java
// How a developer should normally use this
AssetStore<String, ItemQuality, ?> store = ItemQuality.getAssetStore();
ItemQuality rareQuality = store.get("Rare");

if (rareQuality != null) {
    String texturePath = rareQuality.getSlotTexture();
    // Use texturePath to configure UI rendering
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ItemQuality()`. This creates an unmanaged, uninitialized object that is disconnected from the asset system. It will not be recognized by other game systems and will likely cause NullPointerExceptions.
-   **Post-Load Modification:** Do not call setters or modify fields on an instance retrieved from the **AssetStore**. This corrupts the global state for all threads.
-   **Early Access:** Do not attempt to call `getAssetStore()` before the server's asset loading phase is complete. This can result in an empty or incomplete registry.

## Data Pipeline

The ItemQuality class is a key component in the data flow from configuration files to the game client.

> **Ingestion Flow:**
> JSON file on disk (e.g., `assets/hytale/data/definitions/items/qualities/rare.json`) -> Server Asset Loader -> **ItemQuality.CODEC** -> **ItemQuality** instance in **AssetRegistry** memory

> **Egress Flow:**
> Server Game Logic -> **ItemQuality.toPacket()** -> `protocol.ItemQuality` DTO -> Network Serialization Layer -> TCP Packet -> Game Client for Rendering

