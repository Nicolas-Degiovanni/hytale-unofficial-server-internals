---
description: Architectural reference for FieldcraftCategory
---

# FieldcraftCategory

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Data Asset

## Definition
```java
// Signature
public class FieldcraftCategory 
   implements JsonAssetWithMap<String, DefaultAssetMap<String, FieldcraftCategory>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ItemCategory> {
```

## Architecture & Concepts

The FieldcraftCategory class is a server-side data model that represents a single category within the game's crafting system, often referred to as "Fieldcraft". It is not an active service or manager; rather, it is a passive, immutable data structure loaded from configuration files at server startup.

Its primary role is to serve as a strongly-typed representation of a JSON asset. The static `CODEC` field, an instance of AssetBuilderCodec, acts as a blueprint for deserialization. It defines the explicit mapping between keys in the source JSON file (e.g., "Name", "Icon") and the internal fields of the Java object. This codec-driven approach ensures that all FieldcraftCategory assets are loaded, validated, and structured consistently across the entire system.

As an implementation of NetworkSerializable, this class is designed to be transmitted to game clients. The `toPacket` method handles the transformation from this server-side asset representation into a lightweight, optimized network protocol object. This separation ensures that the server's internal data structures are not directly exposed over the network.

All loaded instances are centrally managed and accessed via a static, lazily-initialized asset map, which is populated and owned by the global AssetRegistry. This pattern establishes a single source of truth for all crafting category data on the server.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale AssetRegistry during the server's initial asset loading phase. The public static `CODEC` is used by the registry to instantiate objects via the protected constructor. **WARNING:** Direct manual instantiation is an invalid use case and will result in an uninitialized, non-functional object.
-   **Scope:** Application-scoped. Once loaded at server startup, all FieldcraftCategory instances persist for the entire lifetime of the server process. They are treated as immutable configuration data.
-   **Destruction:** Instances are eligible for garbage collection only upon server shutdown when the AssetRegistry is cleared and the corresponding class loader is unloaded.

## Internal State & Concurrency

-   **State:** The core data fields (id, name, icon, order) are populated once during asset deserialization and are considered immutable thereafter. The only mutable field is `cachedPacket`, a SoftReference used for performance caching. A SoftReference allows the garbage collector to reclaim the cached packet object under memory pressure, preventing potential memory leaks while providing an effective caching mechanism.
-   **Thread Safety:** The class is thread-safe for all read operations. The `toPacket` method, which performs a check-then-act on the `cachedPacket` field, is not synchronized. This introduces a benign race condition where multiple threads could simultaneously generate a new packet object. However, since the source data is immutable, the generated packets will be identical. This is an intentional design choice to favor high-throughput reads without the overhead of locking.

    **WARNING:** The lazy initialization within the static `getAssetMap` method is not thread-safe. It is designed to be called during the server's single-threaded startup sequence. Calling this method from multiple threads concurrently before it has been initialized is unsupported and will lead to unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetMap() | static DefaultAssetMap | O(1) amortized | Retrieves the global, read-only map of all loaded FieldcraftCategory assets. |
| toPacket() | com.hypixel.hytale.protocol.ItemCategory | O(1) | Converts the asset into its network-serializable protocol object, utilizing an internal cache. |
| getId() | String | O(1) | Returns the unique identifier for the category. |
| getName() | String | O(1) | Returns the display name for the category. |
| getIcon() | String | O(1) | Returns the asset path for the category's icon. |

## Integration Patterns

### Standard Usage

FieldcraftCategory objects should not be managed directly. The standard pattern is to retrieve the global asset map and query it for the desired category by its unique string identifier.

```java
// Retrieve the central map of all categories
DefaultAssetMap<String, FieldcraftCategory> categories = FieldcraftCategory.getAssetMap();

// Look up a specific category by its ID (e.g., "hytale:tools")
FieldcraftCategory toolsCategory = categories.get("hytale:tools");

if (toolsCategory != null) {
    // Convert to a network packet to send to a client
    com.hypixel.hytale.protocol.ItemCategory packet = toolsCategory.toPacket();
    // networkService.send(player, packet);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new FieldcraftCategory()`. The constructor is protected and intended for use only by the asset loading system. Manually created instances will be uninitialized and bypass the central AssetRegistry.
-   **State Modification:** Do not attempt to modify the state of a FieldcraftCategory object after it has been loaded, for example, by using reflection. The game server relies on this data being immutable.
-   **Pre-Initialization Access:** Do not attempt to call `getAssetMap` from a system that runs before the main AssetRegistry has completed its loading phase.

## Data Pipeline

The FieldcraftCategory class is a key component in the data pipeline that transforms on-disk configuration into network-ready game data.

> Flow:
> JSON Asset on Disk -> AssetRegistry Deserializer -> **FieldcraftCategory Instance** -> `toPacket()` method call -> Network Protocol Object -> Server Network Layer -> Game Client

