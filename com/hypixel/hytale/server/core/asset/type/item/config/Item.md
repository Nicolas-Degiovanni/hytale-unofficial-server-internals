---
description: Architectural reference for the Item asset configuration class.
---

# Item

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Asset Configuration Object

## Definition
```java
// Signature
public class Item implements JsonAssetWithMap<String, DefaultAssetMap<String, Item>>, NetworkSerializable<ItemBase> {
```

## Architecture & Concepts

The Item class is the definitive, server-side representation of an item's static properties and behaviors. It is not an instance of an item in a player's inventory, but rather the immutable template or "blueprint" from which all item instances derive their characteristics.

This class acts as a bridge between the asset configuration system (JSON files on disk) and the runtime game logic. Its primary architectural roles are:

1.  **Data Schema Definition:** The class, through its static **CODEC_BUILDER**, declaratively defines the complete schema for an item asset. This includes all properties, from visual aspects like models and icons to gameplay mechanics like weapon stats, tool behaviors, and interactions. This builder-based codec approach supports complex features like property inheritance between item definitions.

2.  **Configuration Registry:** All loaded Item definitions are stored in a static, globally accessible **ASSET_STORE**. This serves as a central, in-memory database that server systems query to retrieve an item's blueprint using its unique string identifier.

3.  **Network Serialization Gateway:** Through the NetworkSerializable interface, an Item object can be transformed into a lean, network-optimized Data Transfer Object (DTO) called **ItemBase**. This is a critical performance pattern, ensuring that the rich, server-only configuration data is not wastefully sent over the network. The client receives only the data it needs to render and interact with the item.

The lifecycle of an Item is tied directly to the server's startup and shutdown, not to gameplay events. These objects are loaded once and read from constantly throughout the server's operation.

### Lifecycle & Ownership
-   **Creation:** Item objects are instantiated exclusively by the Hytale **AssetStore** framework during server bootstrap. The static **Item.CODEC** is used to deserialize JSON asset files into memory, populating new Item instances. The **processConfig** method is invoked as a final step in this process to compute derived data and finalize the object's state.

-   **Scope:** An Item object's scope is the entire application. Once loaded, it persists for the full duration of the server session, residing within the static **ASSET_STORE**.

-   **Destruction:** All Item objects are eligible for garbage collection only upon server shutdown when the **AssetRegistry** is cleared.

## Internal State & Concurrency
-   **State:** An Item object is mutable only during the brief asset-loading phase. After the **processConfig** lifecycle method completes, it becomes **effectively immutable**. All collections, such as the *interactions* map, are replaced with unmodifiable versions. The only exception is the transient *cachedPacket* field, a **SoftReference** to a generated **ItemBase** packet, which serves as a memory-sensitive cache.

-   **Thread Safety:** The class is **not thread-safe for mutation**. However, because its state is finalized at startup, it is safe for concurrent reads from multiple threads. Game logic should **never** attempt to modify an Item object retrieved from the asset store.

    **Warning:** While read operations are safe, the caching within the **toPacket** method is not protected by a lock. In a scenario of extremely high contention, this could result in multiple threads redundantly generating the same **ItemBase** packet. This is a benign race condition that prioritizes performance over strict cache atomicity.

## API Surface

The public API is primarily composed of the static asset store accessor and methods for data retrieval and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the central registry containing all loaded Item assets. This is the primary entry point for accessing item data. |
| toPacket() | ItemBase | O(N) | Converts this configuration object into a network-optimized **ItemBase** DTO. Complexity is proportional to the number of complex properties (e.g., particles, interactions) that require conversion. The result is cached via a SoftReference. |
| getId() | String | O(1) | Returns the unique, human-readable identifier for this item asset (e.g., "hytale:stone_sword"). |
| getWeapon() | ItemWeapon | O(1) | Returns the weapon configuration component, or null if this item is not a weapon. |
| getTool() | ItemTool | O(1) | Returns the tool configuration component, or null if this item is not a tool. |
| hasBlockType() | boolean | O(1) | Returns true if this item is directly associated with a placeable block. |

## Integration Patterns

### Standard Usage

All server-side systems must retrieve Item definitions from the central asset map. Never store a direct reference if a string ID will suffice.

```java
// Correct: Retrieve the immutable item definition from the central registry.
DefaultAssetMap<String, Item> itemMap = Item.getAssetMap();
Item stoneSwordConfig = itemMap.getAsset("hytale:stone_sword");

if (stoneSwordConfig != null && stoneSwordConfig.getWeapon() != null) {
    float damage = stoneSwordConfig.getWeapon().getDamage();
    // ... use damage value in combat calculation
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new Item()`. This creates a blank, uninitialized, and invalid object that is not registered with the game systems. It will cause NullPointerExceptions and inconsistent behavior.

-   **State Mutation:** Do not modify an Item object after retrieving it from the asset store. This violates the "effectively immutable" contract and will lead to unpredictable, server-wide state corruption.

-   **Premature Access:** Do not attempt to resolve Item assets while other assets are still being loaded. The **processConfig** method relies on other registries (like SoundEvent, ItemQuality) being fully populated. Accessing items before the asset loading phase is complete may yield partially initialized objects.

## Data Pipeline

The Item class is a key component in the data flow from disk configuration to client-side representation.

> Flow:
> JSON File (`.json`) -> AssetStore Loader -> **Item.CODEC** -> **Item** (Instance in Memory) -> **processConfig()** -> Stored in **Item.ASSET_STORE** -> Game Logic requests Item -> **toPacket()** -> ItemBase (DTO) -> Network Layer -> Client Application

