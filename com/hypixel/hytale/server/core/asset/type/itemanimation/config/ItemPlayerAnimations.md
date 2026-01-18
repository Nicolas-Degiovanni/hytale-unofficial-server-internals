---
description: Architectural reference for ItemPlayerAnimations
---

# ItemPlayerAnimations

**Package:** com.hypixel.hytale.server.core.asset.type.itemanimation.config
**Type:** Asset Type / Data Model

## Definition
```java
// Signature
public class ItemPlayerAnimations
   implements JsonAssetWithMap<String, DefaultAssetMap<String, ItemPlayerAnimations>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ItemPlayerAnimations> {
```

## Architecture & Concepts

The ItemPlayerAnimations class is a data-driven asset definition that encapsulates all animation properties related to how a player character holds and interacts with an item. It is not a service or manager, but rather a configuration object loaded from JSON files at runtime.

This class is a central component of the Hytale **Asset System**. Its primary role is to serve as a structured representation of animation data defined by content creators. The system relies on a powerful `AssetBuilderCodec` for deserialization, which supports an inheritance model. This allows designers to create base animation sets (e.g., a "Default" sword animation) and have specific items (e.g., "Katana") inherit from and override specific properties, promoting reusability and reducing data duplication.

Crucially, this class implements `NetworkSerializable`. This signifies its role as a bridge between server-side asset configuration and client-side rendering. When a player equips an item, the server-side instance of ItemPlayerAnimations is converted into a network packet and synchronized with the client, ensuring the client has the correct animation data to display.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale `AssetStore` framework during the asset loading phase. The static `CODEC` field dictates how JSON files are parsed and materialized into ItemPlayerAnimations objects. **WARNING:** Manual instantiation by developers is an anti-pattern and will bypass the asset management and inheritance systems.

-   **Scope:** An instance of ItemPlayerAnimations, once loaded, persists for the entire application session (server or client). All loaded instances are managed by a singleton `AssetStore` and are accessible globally via static lookup methods like `getAssetStore`.

-   **Destruction:** Objects are garbage collected when the `AssetStore` is cleared, typically during application shutdown. The internal `cachedPacket` uses a `SoftReference`, meaning the cached network packet may be garbage collected under memory pressure even if the parent ItemPlayerAnimations object persists.

## Internal State & Concurrency

-   **State:** The object's state is **effectively immutable** after its creation by the asset loader. All configuration fields like `animations`, `wiggleWeights`, and `camera` are populated once during deserialization and are not intended to be modified thereafter. The only mutable field is `cachedPacket`, which serves as a performance optimization.

-   **Thread Safety:** The class is **conditionally thread-safe**. All configuration data can be safely read by multiple threads simultaneously. The `toPacket` method, which manages the `cachedPacket` `SoftReference`, is not protected by locks. This can lead to a benign race condition where multiple threads might concurrently generate a new packet object if the cache is empty. However, this does not corrupt state and is an acceptable trade-off for performance, as the final state of the cache will simply be the last packet written.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the singleton asset registry for this type. |
| getAnimations() | Map | O(1) | Returns the unmodifiable map of animation names to animation data. |
| toPacket() | ItemPlayerAnimations | O(1) / O(N) | Converts the asset into a network-ready protocol object. Complexity is O(1) on cache hit, O(N) on cache miss where N is the number of properties to copy. |

## Integration Patterns

### Standard Usage

The intended use is to retrieve a pre-loaded instance from the global `AssetStore` and use it to configure game systems or serialize it for network transmission.

```java
// Retrieve the animation set for a specific item from the global registry
DefaultAssetMap<String, ItemPlayerAnimations> animationMap = ItemPlayerAnimations.getAssetMap();
ItemPlayerAnimations swordAnimations = animationMap.get("hytale:longsword_animations");

if (swordAnimations != null) {
    // Convert the asset to a network packet to send to a client
    com.hypixel.hytale.protocol.ItemPlayerAnimations packet = swordAnimations.toPacket();
    player.getNetworkConnection().send(packet);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ItemPlayerAnimations()`. This bypasses the asset loading pipeline, the inheritance system, and the global registry. Objects created this way will be disconnected from the rest of the engine.

-   **State Modification:** Do not attempt to modify the fields of an ItemPlayerAnimations object after it has been retrieved from the `AssetStore`. The system assumes these objects are immutable post-load.

-   **Assuming Cache Persistence:** Do not write code that assumes `toPacket()` will return the exact same object instance every time. The internal cache uses a `SoftReference` and can be cleared by the garbage collector at any time.

## Data Pipeline

The flow of data for this class begins as a static file and ends as a live object used by the client's rendering engine.

> Flow:
> JSON Asset File -> AssetStore Loader -> **ItemPlayerAnimations (CODEC Deserialization)** -> In-Memory AssetStore Registry -> Game Logic Lookup -> **toPacket()** -> Network Packet -> Client Animation System

