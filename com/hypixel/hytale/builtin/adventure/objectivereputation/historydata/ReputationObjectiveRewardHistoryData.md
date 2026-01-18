---
description: Architectural reference for ReputationObjectiveRewardHistoryData
---

# ReputationObjectiveRewardHistoryData

**Package:** com.hypixel.hytale.builtin.adventure.objectivereputation.historydata
**Type:** Data Object

## Definition
```java
// Signature
public final class ReputationObjectiveRewardHistoryData extends ObjectiveRewardHistoryData {
```

## Architecture & Concepts
The ReputationObjectiveRewardHistoryData class is a specialized data structure that represents a historical record of a reputation-based reward. It is a concrete implementation within the objective and reward history system, designed to capture the specific details of a reputation gain, namely which reputation faction was affected and by how much.

This class is fundamentally a data container, often referred to as a Plain Old Java Object (POJO) or a Data Transfer Object (DTO). Its primary architectural role is to serve as a serializable payload. The static CODEC field is the most critical feature, indicating that this object is designed to be encoded to and decoded from a persistent format, such as a player's save file or a network packet.

By extending ObjectiveRewardHistoryData, it participates in a polymorphic data model where different types of rewards can be stored and processed generically, while still retaining their specific attributes.

## Lifecycle & Ownership
-   **Creation:** An instance is created under two circumstances:
    1.  By the game's reward logic when a player completes an objective that grants reputation. The two-argument constructor `ReputationObjectiveRewardHistoryData(String, int)` is used in this case.
    2.  By the Hytale serialization framework during deserialization (e.g., loading a game). The static CODEC field orchestrates this by invoking the default constructor and then populating the fields.

-   **Scope:** The object's lifetime is tied to the collection that holds it. It is a transient object with no independent lifecycle. For example, if it is part of a player's profile, it will exist in memory as long as that profile is loaded.

-   **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for destruction once they are no longer referenced by any part of the game state, such as when a player session ends or a historical log is cleared.

## Internal State & Concurrency
-   **State:** The class holds mutable state consisting of a `reputationGroupId` (String) and an `amount` (int). While the fields can be modified internally (as seen in the codec's lambda functions), the public API only exposes getters. This implies an intent for the object to be treated as immutable after its initial creation or deserialization.

-   **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal locking or synchronization. All operations on an instance should be confined to a single thread, typically the main game thread, to prevent data corruption and race conditions.

## API Surface
The public contract is minimal, focusing on data access and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(1) | Static field used by the engine to serialize and deserialize instances of this class. |
| getReputationGroupId() | String | O(1) | Returns the unique identifier for the reputation group that was affected. |
| getAmount() | int | O(1) | Returns the amount of reputation that was awarded. |

## Integration Patterns

### Standard Usage
This class is not typically used directly by game logic developers. Instead, it is created by the objective system and consumed by systems that need to read historical data, such as UI components that display reward history. The primary interaction mechanism is via the engine's codec system for persistence.

```java
// Example of the system creating a new record
// This would happen deep within the objective completion logic.

String factionId = "hytale_guardians";
int reputationGained = 50;
ObjectiveRewardHistoryData rewardRecord = new ReputationObjectiveRewardHistoryData(factionId, reputationGained);

playerProfile.getHistory().addReward(rewardRecord);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Default Instantiation:** Do not call `new ReputationObjectiveRewardHistoryData()`. This parameterless constructor exists solely for use by the serialization framework. Manually created instances will be in an invalid, partially-initialized state.
-   **State Mutation:** Do not attempt to modify the state of this object via reflection after it has been created. It is designed to be a static record of a past event.

## Data Pipeline
ReputationObjectiveRewardHistoryData acts as a data payload that flows through the game's state management and persistence pipelines.

> **Reward Granting Flow:**
> Objective Completion Event -> Reward Granting Service -> **new ReputationObjectiveRewardHistoryData(id, amount)** -> Player Profile State -> Serializer (using CODEC) -> Player Save File

> **Game Load Flow:**
> Player Save File -> Deserializer (using CODEC) -> **ReputationObjectiveRewardHistoryData instance** -> Player Profile State -> UI or Game Logic Read Access

