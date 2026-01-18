---
description: Architectural reference for ItemSoundSet
---

# ItemSoundSet

**Package:** com.hypixel.hytale.server.core.asset.type.itemsound.config
**Type:** Managed Asset

## Definition
```java
// Signature
public class ItemSoundSet
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, ItemSoundSet>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ItemSoundSet> {
```

## Architecture & Concepts
The ItemSoundSet class is a data definition object that represents a collection of sound events associated with an item, such as sounds for hitting, breaking, or using the item. It is a critical component of the Hytale Asset System, acting as the in-memory representation of a corresponding JSON configuration file.

Its primary architectural role is to serve as an immutable, optimized bridge between human-readable asset identifiers (e.g., a string like "hytale:sound.item.sword_hit") and a network-efficient, integer-based index format used for server-client communication.

This transformation is defined declaratively by the static **CODEC** field. The codec orchestrates the entire deserialization process:
1.  Reading the JSON data into the object's fields.
2.  Executing a series of validators to ensure data integrity and referential correctness (e.g., verifying that the referenced SoundEvent assets exist).
3.  Triggering the **processConfig** post-processing step to compute the optimized integer indices from the raw string identifiers.

The class implements NetworkSerializable, signaling its role in the data contract between the server and the game client. The **toPacket** method handles the final conversion to a lightweight protocol buffer object for network transmission.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale **AssetStore** during the server's bootstrap or asset-loading phase. The static **CODEC** is used to deserialize JSON asset files into ItemSoundSet objects. Direct instantiation by developers is an anti-pattern and will result in a non-functional object.
-   **Scope:** An ItemSoundSet instance is a flyweight. A single, shared instance exists for each unique sound set defined in the game's assets. Once loaded, it persists for the entire lifetime of the server process.
-   **Destruction:** Instances are garbage collected when the server shuts down and the central AssetRegistry is cleared.

## Internal State & Concurrency
-   **State:** The object is effectively immutable after initialization. Its state is established once during asset loading and is not intended to be modified at runtime.
    -   **soundEventIds:** A map of enum-to-string, representing the raw configuration loaded from JSON.
    -   **soundEventIndices:** A transient, derived map of enum-to-integer. This is an optimized lookup table for runtime use, computed by **processConfig**.
    -   **cachedPacket:** A SoftReference to the generated network packet. This is a performance cache to avoid repeated object allocation and field population.

-   **Thread Safety:** The class is thread-safe for all read operations after the initial asset loading phase is complete. The internal state is not mutated after the **processConfig** method is called by the asset loader.

    **Warning:** The **toPacket** method's internal caching mechanism is not strictly atomic. In a highly concurrent scenario where multiple threads call it on a new instance simultaneously, the packet might be generated more than once. However, this is a benign race condition and is considered acceptable within the engine's threading model.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, singleton asset store for all ItemSoundSet assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Retrieves the asset map for efficient ID-to-instance and index-to-instance lookups. |
| toPacket() | com.hypixel.hytale.protocol.ItemSoundSet | O(1) Amortized | Converts the asset into its network-ready protocol buffer representation. Uses an internal cache for high performance. |
| getId() | String | O(1) | Returns the unique string identifier for this asset (e.g., "hytale:wood_sword_sounds"). |
| getSoundEventIds() | Map | O(1) | Returns the raw mapping of sound events to their string-based asset keys. **Warning:** Do not modify this map. |
| getSoundEventIndices() | Object2IntMap | O(1) | Returns the optimized mapping of sound events to their integer-based network indices. |

## Integration Patterns

### Standard Usage
An ItemSoundSet should never be instantiated directly. It must be retrieved from the central asset system using its unique string identifier.

```java
// Correct: Retrieve the managed instance from the asset map.
IndexedLookupTableAssetMap<String, ItemSoundSet> map = ItemSoundSet.getAssetMap();
ItemSoundSet swordSounds = map.get("hytale:wood_sword_sounds");

if (swordSounds != null) {
    // Convert to a network packet to send to a client.
    com.hypixel.hytale.protocol.ItemSoundSet packet = swordSounds.toPacket();
    // networkManager.send(player, packet);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** `new ItemSoundSet()` is strictly forbidden. This bypasses the asset loading pipeline, including validation and the critical **processConfig** step. The resulting object will lack its optimized index map and will be invalid.
-   **State Mutation:** The maps returned by **getSoundEventIds** and **getSoundEventIndices** must be treated as read-only. Modifying them will lead to undefined behavior and desynchronization across the server.

## Data Pipeline
The ItemSoundSet class is a key stage in the asset processing pipeline, transforming declarative configuration into an optimized, runtime-ready format.

> Flow:
> 1. **ItemSoundSet.json** (On-disk asset file)
> 2. **AssetStore** (Deserializes the JSON using the static CODEC)
> 3. **ItemSoundSet Instance** (In-memory object with raw string IDs)
> 4. **processConfig()** (Internal post-processing to map string IDs to integer indices)
> 5. **toPacket()** (Conversion to a network DTO)
> 6. **Network Layer** (Transmits the optimized packet to the game client)

