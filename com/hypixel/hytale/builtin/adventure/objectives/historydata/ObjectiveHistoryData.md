---
description: Architectural reference for ObjectiveHistoryData
---

# ObjectiveHistoryData

**Package:** com.hypixel.hytale.builtin.adventure.objectives.historydata
**Type:** Transient

## Definition
```java
// Signature
public final class ObjectiveHistoryData extends CommonObjectiveHistoryData {
```

## Architecture & Concepts
The ObjectiveHistoryData class is a specialized data container responsible for tracking the state and outcomes of a single game objective, particularly the rewards granted to individual players. It functions as a Data Transfer Object (DTO) within the adventure mode systems, designed for both server-side state management and network serialization.

This class extends CommonObjectiveHistoryData, inheriting baseline properties such as an objective ID and category. Its primary architectural role is to aggregate reward data on the server and then produce player-specific, sanitized snapshots for transmission to clients.

The static CODEC field indicates that this class is a core component of the engine's data persistence and networking layer. It is designed to be robustly serialized to and from a binary or text format, ensuring data integrity across network boundaries or between game sessions.

The internal design distinguishes between two states:
1.  A **master state** (on the server) which uses the `rewardsPerPlayer` map to track rewards for all participating players.
2.  A **snapshot state** (on the client or for a specific player) which uses the `rewards` array to hold a filtered list of rewards relevant only to that player. The `cloneForPlayer` method is the bridge between these two states.

## Lifecycle & Ownership
-   **Creation:** Instances are typically created by a higher-level objective management service when an objective becomes active. They are also instantiated by the framework's deserialization pipeline (via the CODEC) when loading game state or receiving network packets. The `cloneForPlayer` method is a secondary factory for creating player-specific instances.
-   **Scope:** The lifetime of an ObjectiveHistoryData object is tied to the objective it represents. A master instance may persist on the server for the duration of a world or zone. Cloned, player-specific instances are often short-lived, existing only for the duration of a network transmission and subsequent client-side processing.
-   **Destruction:** As a standard Java object, it is managed by the garbage collector. There are no explicit cleanup or disposal methods. Instances are eligible for collection once they are no longer referenced by the objective system, player profiles, or network serializers.

## Internal State & Concurrency
-   **State:** The object is highly mutable. Methods like `addRewardForPlayerUUID` and `completed` directly alter its internal collections. Its state transitions from an open, aggregating phase to a finalized, snapshot phase.
-   **Thread Safety:** This class is designed for concurrent access. The `rewardsPerPlayer` field is a ConcurrentHashMap, making the `addRewardForPlayerUUID` operation thread-safe. This is critical in a server environment where multiple player threads might grant rewards for the same objective simultaneously. However, the `rewards` array is not thread-safe and should be considered immutable after its creation within the `cloneForPlayer` or `completed` methods.

**WARNING:** Do not modify the array returned by `getRewards`. It is a direct reference to the internal state and is not safe for concurrent modification.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRewards() | ObjectiveRewardHistoryData[] | O(1) | Returns the finalized array of rewards for a specific player. Null if not finalized. |
| addRewardForPlayerUUID(uuid, data) | void | O(1) avg | Atomically adds a reward record for a specific player. Safe to call from multiple threads. |
| cloneForPlayer(uuid) | ObjectiveHistoryData | O(N) | Creates a new, sanitized instance containing only the rewards for the specified player. N is the number of rewards for that player. |
| completed(uuid, data) | void | O(N) | Finalizes the object's state, typically after an objective is fully completed. Copies rewards from a source object for the specified player. |

## Integration Patterns

### Standard Usage
This class is not intended to be managed directly. It is manipulated by the core objective and quest systems. The primary pattern involves the server maintaining a master instance, adding rewards as players earn them, and then using `cloneForPlayer` to create a snapshot to send to the client upon completion.

```java
// Server-side objective service logic (conceptual)
ObjectiveHistoryData masterHistory = objectiveService.getHistoryFor("main_quest_1");

// Multiple threads can call this concurrently as players earn rewards
masterHistory.addRewardForPlayerUUID(player.getUUID(), new ObjectiveRewardHistoryData(...));

// When sending an update to a specific player
ObjectiveHistoryData playerSnapshot = masterHistory.cloneForPlayer(player.getUUID());
networkManager.sendToClient(player, playerSnapshot);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid `new ObjectiveHistoryData()`. The objective system or deserialization framework is responsible for creating and managing these objects. Manual creation can bypass necessary initialization and lead to an invalid state.
-   **State Modification after Cloning:** Do not call `addRewardForPlayerUUID` on a master object after `cloneForPlayer` has been used to generate a final state for a player. This can create a discrepancy between the server's view and what the client receives.
-   **Client-Side Modification:** Client code should treat received ObjectiveHistoryData instances as read-only. The data represents a historical record and should not be modified.

## Data Pipeline
ObjectiveHistoryData serves as a payload in the objective completion data flow.

> **Server Flow:**
> Game Event (e.g., monster defeated) → Objective Service → `addRewardForPlayerUUID` on master **ObjectiveHistoryData** → Objective Completion Trigger → `cloneForPlayer` → Player-specific **ObjectiveHistoryData** instance → Serializer (CODEC) → Network Packet

> **Client Flow:**
> Network Packet → Deserializer (CODEC) → Player-specific **ObjectiveHistoryData** instance → UI System (e.g., Quest Complete screen) → `getRewards` → Display Rewards

