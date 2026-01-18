---
description: Architectural reference for ItemObjectiveRewardHistoryData
---

# ItemObjectiveRewardHistoryData

**Package:** com.hypixel.hytale.builtin.adventure.objectives.historydata
**Type:** Transient / Data Transfer Object

## Definition
```java
// Signature
public final class ItemObjectiveRewardHistoryData extends ObjectiveRewardHistoryData {
```

## Architecture & Concepts
ItemObjectiveRewardHistoryData is a specialized data container that represents a historical record of an item-based reward granted to a player for completing an objective. It is not a service or manager, but rather a passive, serializable data structure.

Its primary architectural role is to serve as a concrete implementation within a polymorphic reward system. The parent class, ObjectiveRewardHistoryData, likely defines a common interface for various reward types, allowing the game engine to process them generically.

The most critical feature of this class is the static **CODEC** field. This self-contained serialization and deserialization logic allows the engine to reliably convert instances of this class to and from persistent storage formats (e.g., game saves, server state snapshots) or network packets. The codec ensures data integrity by defining the exact keys ("ItemId", "Quantity") and data types, and by integrating with the Item validator cache to ensure the referenced item ID is valid.

## Lifecycle & Ownership
-   **Creation:** Instances are created through two primary mechanisms:
    1.  **Deserialization:** The static CODEC instantiates the object when loading data from a persistent source, such as a player's save file. This is the most common creation path.
    2.  **Direct Instantiation:** The game's objective logic may create a new instance programmatically via `new ItemObjectiveRewardHistoryData(itemId, quantity)` when an objective is completed and its reward is being recorded for the first time.

-   **Scope:** The object's lifetime is bound to the data it represents. It is a transient object, typically held within a collection as part of a player's profile or objective history. Multiple instances can and will exist simultaneously for different rewards. It does not persist for the entire application session unless held by a session-scoped object.

-   **Destruction:** The object is eligible for garbage collection as soon as all references to it are released. This typically occurs when a player session ends or when historical data is purged.

## Internal State & Concurrency
-   **State:** The internal state is **Mutable** but should be treated as immutable after initial construction. It consists of two fields: `itemId` and `quantity`. The class is designed to be a simple data holder with no internal logic that modifies its own state.

-   **Thread Safety:** This class is **Not Thread-Safe**. It is a plain data object with no synchronization primitives. All operations, including creation via the codec and read access via getters, must be performed on a single, controlled thread, such as the main server tick thread. Unsynchronized access from multiple threads will lead to race conditions and undefined behavior.

## API Surface
The public contract is minimal, focusing on data access and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(N) | **Critical.** Static field for serializing and deserializing instances. Complexity is proportional to the number of fields. |
| getItemId() | String | O(1) | Returns the unique identifier for the item that was rewarded. |
| getQuantity() | int | O(1) | Returns the number of items that were rewarded. |

## Integration Patterns

### Standard Usage
This class is not meant to be used in isolation. It is consumed by higher-level systems, such as an objective manager or a player profile service, which use its CODEC to deserialize reward data and then read its properties to grant items to the player.

```java
// Example: A hypothetical system processing a list of rewards
// Note: The object is typically obtained via deserialization, not direct creation.

List<ObjectiveRewardHistoryData> rewards = playerProfile.getObjectiveRewards();

for (ObjectiveRewardHistoryData reward : rewards) {
    if (reward instanceof ItemObjectiveRewardHistoryData) {
        ItemObjectiveRewardHistoryData itemReward = (ItemObjectiveRewardHistoryData) reward;
        playerInventory.addItem(itemReward.getItemId(), itemReward.getQuantity());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not modify the internal fields of this object after it has been created. It represents a historical fact and should be treated as immutable. Modifying it post-creation can lead to data desynchronization between game state and persistent storage.
-   **Manual Serialization:** Never attempt to serialize or deserialize this object manually (e.g., by writing your own JSON parser for it). The provided static CODEC is the single source of truth for its data format and includes crucial validation logic. Bypassing it will lead to data corruption and compatibility issues.

## Data Pipeline
As a data object, ItemObjectiveRewardHistoryData is the *payload* within a data pipeline, not an active component of it. The pipeline describes how data is transformed into an instance of this class.

> Flow:
> Game Save File / Network Packet -> Engine's Codec Deserializer -> **ItemObjectiveRewardHistoryData Instance** -> Objective Processing System -> Player Inventory Update

