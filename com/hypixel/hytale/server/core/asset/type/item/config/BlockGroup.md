---
description: Architectural reference for BlockGroup
---

# BlockGroup

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Data Asset

## Definition
```java
// Signature
@Deprecated
public class BlockGroup implements JsonAssetWithMap<String, DefaultAssetMap<String, BlockGroup>>, NetworkSerializable<com.hypixel.hytale.protocol.BlockGroup> {
```

## Architecture & Concepts

The BlockGroup class is a data-driven asset that represents a named collection of block identifiers. Its primary function is to logically group different blocks together, which can then be used by other game systems for logic such as creative inventory organization, tool effectiveness, or world generation rules.

Architecturally, BlockGroup instances are not managed directly by developers. They are defined in external JSON files and loaded into memory at server startup by the AssetRegistry. The class acts as the in-memory representation of these configuration files.

The static `CODEC` field is central to its design, defining how a JSON object is deserialized into a BlockGroup instance. This integration with the asset pipeline makes it a fundamental, read-only data structure for the server.

**WARNING:** This class is marked as **Deprecated**. New development should avoid using it. Its functionality is likely superseded by a more robust or performant system. Existing usage should be considered for migration.

### Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the asset loading system via the `AssetBuilderCodec` defined in the static `CODEC` field. The codec populates the instance fields directly from parsed JSON data. Manual instantiation is an anti-pattern and will result in a non-functional object.
-   **Scope:** A BlockGroup instance, once loaded, persists for the entire server session. It is stored within the `AssetStore` managed by the `AssetRegistry` and is considered static configuration data.
-   **Destruction:** Instances are destroyed when the server shuts down or initiates a full asset reload, at which point the managing `AssetStore` is cleared.

## Internal State & Concurrency

-   **State:** The internal state, particularly the `blocks` array, is mutable only during the asset deserialization process. After being fully loaded and registered, instances should be treated as **effectively immutable**. Modifying a loaded BlockGroup at runtime is unsupported and will lead to unpredictable behavior.
-   **Thread Safety:** This class is **not thread-safe**. Its methods do not employ any synchronization mechanisms. However, as it is intended to be read-only data after initialization, it can be safely accessed from the main server thread.

    **WARNING:** The static `findItemGroup` method iterates over the global asset map. Calling this method from multiple threads concurrently without external synchronization on the `AssetRegistry` can lead to `ConcurrentModificationException` or other race conditions if assets are being reloaded.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| findItemGroup(Item) | static BlockGroup | O(N\*M) | Scans all registered BlockGroups to find one containing the item's block ID. **High performance cost.** |
| getId() | String | O(1) | Returns the unique identifier for this group, derived from its asset name. |
| get(int) | String | O(1) | Retrieves a block ID by its index within the group. |
| size() | int | O(1) | Returns the total number of block IDs in this group. |
| getIndex(Item) | int | O(N) | Performs a linear search for the item's block ID within this group. Returns -1 if not found. |
| toPacket() | protocol.BlockGroup | O(N) | Serializes the group's data into a network-transferable packet object. |

## Integration Patterns

### Standard Usage

The most common interaction is to determine which group a specific item belongs to. This is accomplished via the static `findItemGroup` utility method.

```java
// Given an Item instance, find its corresponding BlockGroup
Item myItem = ...;
BlockGroup group = BlockGroup.findItemGroup(myItem);

if (group != null) {
    // Logic that depends on the item's group
    System.out.println("Item belongs to group: " + group.getId());
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance with `new BlockGroup()`. The object will be uninitialized and lack an ID and block list. All instances must be loaded from JSON assets.
-   **Frequent Lookups:** Avoid calling `findItemGroup` repeatedly in performance-critical code like the main game loop. Its complexity is high, as it iterates through all groups and their contents. Cache the result if it is needed for multiple operations on the same item.
-   **Runtime Modification:** Do not attempt to modify the `blocks` array of a loaded BlockGroup. This violates the immutable-after-load contract of the asset system and can cause desynchronization between game logic and the network protocol.

## Data Pipeline

The BlockGroup class is primarily populated through the server's asset loading pipeline and can be serialized for network transmission.

> **Asset Loading Flow:**
> JSON File on Disk -> AssetRegistry Loader -> `AssetBuilderCodec` -> **BlockGroup Instance** -> In-Memory AssetStore

> **Network Serialization Flow:**
> Game Logic Request -> **BlockGroup Instance** -> `toPacket()` Method -> `protocol.BlockGroup` -> Network Encoder -> Client
---

