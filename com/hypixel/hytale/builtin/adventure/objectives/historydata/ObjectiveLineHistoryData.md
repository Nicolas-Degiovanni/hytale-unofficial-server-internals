---
description: Architectural reference for ObjectiveLineHistoryData
---

# ObjectiveLineHistoryData

**Package:** com.hypixel.hytale.builtin.adventure.objectives.historydata
**Type:** Transient Data Model

## Definition
```java
// Signature
public final class ObjectiveLineHistoryData extends CommonObjectiveHistoryData {
```

## Architecture & Concepts
The ObjectiveLineHistoryData class is a specialized data container that represents the historical state of a single "objective line" within the adventure mode quest system. An objective line is a sequence or group of individual objectives that must be completed together. This class acts as a composite, aggregating multiple ObjectiveHistoryData instances into a single, serializable unit.

Its primary architectural role is to serve as a data transfer object (DTO) for persistence and state tracking. The presence of a static CODEC field indicates that its main purpose is to be serialized to disk for game saves or potentially transmitted over the network.

This class embodies two key concepts:
1.  **Template State:** A master ObjectiveLineHistoryData object can define the structure of a quest line.
2.  **Per-Player Instance State:** The class provides a mechanism (cloneForPlayers) to stamp out unique, stateful copies for each player actively engaged in the quest, ensuring player progress is tracked independently.

## Lifecycle & Ownership
-   **Creation:** Instances are created through two primary pathways:
    1.  **Deserialization:** The most common path. An instance is created and hydrated with data by the Hytale CODEC system when loading a world save or player profile. The private no-argument constructor exists solely for this purpose.
    2.  **Programmatic Instantiation:** The quest engine may create new instances to represent a player's initial progress on a quest line or via the cloneForPlayers method to create derived, per-player state.

-   **Scope:** The lifetime of an ObjectiveLineHistoryData instance is tied to the player or world data it represents. It is not a singleton or a long-lived service. It persists as part of a player's session data or within the world's quest state graph and is discarded when that data is unloaded.

-   **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection when no longer referenced by the active quest system, such as when a player's data is unloaded from memory or a quest is abandoned and its history purged.

## Internal State & Concurrency
-   **State:** This object is **highly mutable**. Methods such as addObjectiveHistoryData and completed directly modify the internal array of objectives. It is designed to be a stateful record of progress that is updated over time.

-   **Thread Safety:** **WARNING:** This class is **not thread-safe**. Its internal collections are not synchronized, and mutation methods are not protected against concurrent access. All operations on an ObjectiveLineHistoryData instance must be performed on a single, controlled thread, typically the main server game thread. Failure to do so will result in race conditions, inconsistent state, and data corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addObjectiveHistoryData(data) | void | O(N) | Appends a new objective's history to the internal array. Complexity is due to array copying. |
| cloneForPlayers(playerUUIDs) | Map | O(M * P) | Factory method. Creates a deep, per-player copy of the objective line state for a set of players. M = objectives, P = players. |
| completed(playerUUID, data) | void | O(M * S) | Merges completion data from a new instance into this one for a specific player. M = new objectives, S = saved objectives. |

## Integration Patterns

### Standard Usage
The primary pattern involves loading an instance, updating it with new progress, and allowing it to be saved back to disk by the engine. The completed method is the main entry point for state mutation.

```java
// Assume 'player' and 'questSystem' are in scope
// 1. Retrieve the player's current quest history
ObjectiveLineHistoryData savedHistory = questSystem.getHistoryFor(player, "main_quest_line_01");

// 2. A new update arrives (e.g., from a completed objective trigger)
ObjectiveLineHistoryData latestUpdate = createUpdateFromTrigger(); // Contains one completed objective

// 3. Merge the new progress into the saved history
savedHistory.completed(player.getUUID(), latestUpdate);

// The questSystem will handle serializing the modified 'savedHistory' during the next save cycle.
```

### Anti-Patterns (Do NOT do this)
-   **Shared State Mutation:** Do not retrieve a single ObjectiveLineHistoryData instance and pass it to multiple player threads or systems. Modifying this shared instance will cause progress from one player to incorrectly apply to all others. Always use cloneForPlayers to create distinct copies.
-   **Asynchronous Modification:** Do not modify an instance from a network thread or a separate worker thread. All calls to methods like completed or addObjectiveHistoryData must be marshaled back to the main game thread.

## Data Pipeline
This class is a payload within a larger data persistence and state management pipeline. It does not actively process data but is instead the data being processed.

> Flow:
> Game Save on Disk -> Hytale Codec System (Deserialization) -> **ObjectiveLineHistoryData Instance** -> Quest Engine (State Merging via completed) -> Modified **ObjectiveLineHistoryData Instance** -> Hytale Codec System (Serialization) -> Game Save on Disk

