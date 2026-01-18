---
description: Architectural reference for CommonObjectiveHistoryData
---

# CommonObjectiveHistoryData

**Package:** com.hypixel.hytale.builtin.adventure.objectives.historydata
**Type:** Data Model / Base DTO

## Definition
```java
// Signature
public abstract class CommonObjectiveHistoryData {
```

## Architecture & Concepts
CommonObjectiveHistoryData is the abstract base class for all objective history records within the adventure mode system. It serves as a foundational data structure, not a service, designed to track a player's progress and completion history for a specific objective. It encapsulates the universal properties of any objective's history, such as its unique identifier, completion count, and the timestamp of the last completion.

The key architectural feature of this class is its deep integration with Hytale's serialization framework, the Codec system. It defines two critical static fields:
1.  **BASE_CODEC:** A BuilderCodec that defines the serialization and deserialization logic for the common fields shared by all subclasses (id, timesCompleted, etc.).
2.  **CODEC:** A CodecMapCodec which acts as a polymorphic dispatcher. When deserializing data, this codec first reads a "Type" field to identify the specific kind of objective history (e.g., "KillObjective", "LocationObjective"). It then delegates the rest of the deserialization process to the appropriate concrete subclass codec, which will handle any additional, type-specific data.

This design allows the system to manage a heterogeneous collection of objective history records in a type-safe and extensible manner, without the parent system needing to know the details of every possible objective type.

### Lifecycle & Ownership
-   **Creation:** Instances are created in two primary scenarios:
    1.  **Programmatically:** A new concrete subclass instance is instantiated when a player completes a particular objective for the very first time.
    2.  **Deserialization:** The Codec system instantiates objects using the protected no-arg constructor when loading a player's saved game data from disk or receiving it over the network.
-   **Scope:** The object's lifetime is bound to a player's session. It is loaded into memory when the player enters a world and persists until the session ends. It is not a global or long-lived service.
-   **Destruction:** The object is eligible for garbage collection once the player's session data is unloaded. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** This object is highly **mutable**. Its core purpose is to maintain state about player progress. The `completed` method directly modifies the internal `timesCompleted` and `lastCompletionTimestamp` fields.
-   **Thread Safety:** This class is **not thread-safe**. The `completed` method, which performs an increment operation (`timesCompleted++`), is a critical section. Concurrent calls to this method from multiple threads will result in race conditions and data corruption. All mutations must be externally synchronized by the owning system, such as a central `PlayerObjectiveManager`.

## API Surface
The public API consists primarily of getters. The most significant method is the protected mutator.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| completed() | void | O(1) | Increments the completion counter and updates the last completion timestamp to the current time. **Warning:** This operation is not atomic and is not thread-safe. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. Developers interact with its concrete subclasses, typically through a managing service that holds a player's entire objective history. The manager is responsible for retrieving the correct history object and safely invoking state changes.

```java
// Assumes a manager service that handles a player's data
PlayerObjectiveManager manager = player.getObjectiveManager();

// Retrieve a specific objective's history (as a concrete subclass)
KillObjectiveHistoryData skeletonQuestHistory = manager.getHistory("quests.main.kill_10_skeletons", KillObjectiveHistoryData.class);

// Safely update the objective's state via the manager
if (skeletonQuestHistory != null) {
    // The manager is responsible for ensuring this call is thread-safe
    manager.recordCompletion(skeletonQuestHistory); 
}
```

### Anti-Patterns (Do NOT do this)
-   **Unsynchronized Mutation:** Directly calling the `completed` method from multiple threads without an external locking mechanism is the most severe anti-pattern. This will lead to an incorrect `timesCompleted` count.
-   **State Management:** Do not attempt to manage collections of these objects manually. Rely on the designated engine service (e.g., PlayerObjectiveManager) which correctly handles loading, saving, and safe mutation.
-   **Incorrect Deserialization:** Do not attempt to deserialize this object using `BASE_CODEC`. Always use the polymorphic `CODEC` to ensure that subclass-specific data is correctly processed.

## Data Pipeline
This object is a critical payload in the player data persistence pipeline.

> **Serialization Flow (Saving Game):**
> Player Session Unload -> PlayerObjectiveManager -> **CommonObjectiveHistoryData Instance** -> Polymorphic `CODEC` serializes base and subclass data -> Serialized Binary/JSON -> Player Save File

> **Deserialization Flow (Loading Game):**
> Player Save File -> Serialized Binary/JSON -> Polymorphic `CODEC` reads "Type" field -> Dispatches to correct subclass codec -> **CommonObjectiveHistoryData Instance** created and populated -> PlayerObjectiveManager -> Player Session Ready

