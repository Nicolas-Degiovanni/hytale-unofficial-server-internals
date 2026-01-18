---
description: Architectural reference for ItemAttitudeMap
---

# ItemAttitudeMap

**Package:** com.hypixel.hytale.server.npc.blackboard.view.attitude
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class ItemAttitudeMap {
    // Inner Builder class
    public static class Builder { ... }
}
```

## Architecture & Concepts
ItemAttitudeMap is a specialized, performance-oriented data structure that serves as a read-optimized cache for NPC attitudes towards specific items. It is a core component of the NPC AI system, providing a fast lookup mechanism to determine how an NPC should react to an item presented to it or found in the world.

Architecturally, this class sits between the static asset system and the dynamic entity simulation. It pre-processes and flattens complex, tag-based item groupings defined in configuration files into a simple, direct map of item IDs to Attitude enums. This pre-computation, handled by the inner Builder class, is critical for server performance, as it avoids expensive tag resolution and asset lookups during the main game loop.

The map is organized into "attitude groups," allowing different types of NPCs (e.g., a Villager versus a Guard) to have distinct sets of preferences without requiring separate map instances for each. An NPC's role determines which group it uses for lookups.

## Lifecycle & Ownership
- **Creation:** An ItemAttitudeMap is exclusively constructed via its inner `ItemAttitudeMap.Builder`. The builder is typically invoked once during server initialization or world loading by a higher-level service responsible for loading NPC configurations. The builder consumes `ItemAttitudeGroup` assets, resolves all item tags, and populates the final map structure.
- **Scope:** The created instance is long-lived. It is designed to be held by a central NPC management service and persist for the lifetime of the server or world instance. Its state can be modified at runtime via hot-reloading.
- **Destruction:** The object is eligible for garbage collection when the server or world shuts down and the owning service releases its reference. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The primary internal state is a final array of Maps: `Map<String, Attitude>[]`. While the array reference itself is final, the Maps contained within it are mutable and can be replaced entirely by the `updateAttitudeGroup` method. This makes the overall object a mutable container. The data is a cache derived from game asset files.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. The underlying data structure is a raw array of Maps. The `updateAttitudeGroup` method performs a non-atomic write to an array index, while `getAttitude` performs a non-atomic read. Concurrent access can lead to race conditions, where a read operation might access a null or partially constructed map during an update. All access and modification must be externally synchronized or confined to a single thread, such as the main server tick thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAttitude(parent, item) | Attitude | O(1) | Retrieves the configured Attitude for a given item, based on the parent NPC's attitude group. Returns null if no specific attitude is defined. |
| getAttitudeGroupCount() | int | O(1) | Returns the total number of attitude groups configured. |
| updateAttitudeGroup(id, group) | void | O(N*M) | Replaces an entire attitude group at runtime. Complexity is high as it must resolve all item tags within the group. This is intended for configuration hot-reloading. |
| Builder.build() | ItemAttitudeMap | O(C) | Finalizes the construction process and returns the immutable map instance. Complexity depends on the total number of configured attitudes. |

## Integration Patterns

### Standard Usage
The map should be built once during server startup and stored in a central service. AI behavior trees or decision-making systems then query this central instance.

```java
// During server initialization
ItemAttitudeMap.Builder builder = new ItemAttitudeMap.Builder();
builder.addAttitudeGroups(assetService.loadAll(ItemAttitudeGroup.class));
ItemAttitudeMap attitudeMap = builder.build();
npcContext.setAttitudeMap(attitudeMap);

// During an NPC's update tick
void onPlayerGift(NPCEntity npc, ItemStack offeredItem) {
    ItemAttitudeMap attitudeMap = npcContext.getAttitudeMap();
    Attitude reaction = attitudeMap.getAttitude(npc, offeredItem);
    
    if (reaction == Attitude.LIKE) {
        npc.playHappyAnimation();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call `updateAttitudeGroup` from one thread while `getAttitude` might be called from another. This will lead to unpredictable behavior and potential NullPointerExceptions. All modifications must be synchronized.
- **Frequent Rebuilding:** Do not use the Builder to create new maps frequently during gameplay. The Builder is a heavy object designed for one-time setup. For runtime changes, use the `updateAttitudeGroup` method on the existing instance.
- **Direct Instantiation:** The constructor is private. Attempting to instantiate it directly via reflection will result in an uninitialized and unusable object. Always use the `Builder`.

## Data Pipeline
The data flow for this component demonstrates the transformation from human-readable configuration to a runtime-optimized data structure.

> Flow:
> 1. **Asset Files (.json)**: An NPC designer defines an `ItemAttitudeGroup`, using item tags like "hytale:gemstone" to represent categories of items.
> 2. **AssetRegistry**: The server loads these files into `ItemAttitudeGroup` asset objects.
> 3. **ItemAttitudeMap.Builder**: The builder is populated with these asset objects.
> 4. **Tag Resolution**: The builder queries the `Item.getAssetMap()` to expand tags like "hytale:gemstone" into a full set of concrete item IDs ("hytale:diamond", "hytale:ruby", etc.).
> 5. **Map Creation**: The builder creates the final, flattened `Map<String, Attitude>` where each key is a concrete item ID.
> 6. **ItemAttitudeMap**: The final object holds this optimized map.
> 7. **NPC System**: At runtime, the system performs a direct, O(1) lookup using an `ItemStack`'s item ID to retrieve an `Attitude`.

