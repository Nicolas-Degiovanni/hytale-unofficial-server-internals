---
description: Architectural reference for TreasureChestState
---

# TreasureChestState

**Package:** com.hypixel.hytale.builtin.adventure.objectives.blockstates
**Type:** State Object

## Definition
```java
// Signature
public class TreasureChestState extends ItemContainerState implements BreakValidatedBlockState {
```

## Architecture & Concepts
The TreasureChestState is a specialized block state that represents a quest-specific container within the world. It is not a generic chest; instead, it functions as a physical, in-world checkpoint for an adventure mode Objective. Its primary architectural role is to act as a gatekeeper, enforcing game logic by linking a physical block to an abstract quest entity.

This class bridges the gap between the World system and the Objective system. It uses a unique identifier, objectiveUUID, to associate itself with a parent Objective. This link allows it to query the ObjectivePlugin to validate player interactions, ensuring that only players actively participating in the quest can open the chest.

Upon a successful opening, it follows an event-driven pattern by dispatching a TreasureChestOpeningEvent onto the server's main event bus. This decouples the chest's state change from the systems that need to react to it, such as the Objective system that updates quest progress.

## Lifecycle & Ownership
-   **Creation:** TreasureChestState instances are not intended for direct instantiation via a constructor in game logic. They are deserialized from world storage by the world engine using the provided static CODEC. Programmatic creation is reserved for high-level systems, such as world generation or objective setup scripts, which populate the state and prepare it for serialization.
-   **Scope:** The lifecycle of a TreasureChestState object is strictly tied to the physical block it represents in the game world. It persists as long as the block exists in the world save data.
-   **Destruction:** The object is eligible for garbage collection when the world chunk containing its associated block is unloaded from memory. It is permanently destroyed when a player breaks the block in-game, an action which is only permitted after the chest has been opened.

## Internal State & Concurrency
-   **State:** This object is mutable. Its core state is managed by the boolean *opened* flag, which transitions from false to true and dictates interaction logic. It also contains mutable inventory state inherited from ItemContainerState. The objectiveUUID and chestUUID fields are considered immutable after initial setup.
-   **Thread Safety:** **This class is not thread-safe.** All interactions with block states, including TreasureChestState, must be performed on the main server thread responsible for the world tick. The Hytale engine's architecture guarantees this for standard player interactions. Off-thread modification will lead to severe concurrency issues, race conditions, and world corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canOpen(ref, accessor) | boolean | O(N) | Validates if the interacting entity is an active participant in the linked objective. Complexity depends on the lookup time for active players in the Objective. |
| canDestroy(ref, accessor) | boolean | O(1) | Validates if the chest can be broken. Returns true only if the *opened* flag is true. |
| onOpen(ref, world, store) | void | O(N) | Executes logic upon opening. Dispatches TreasureChestOpeningEvent to N listeners and mutates internal state. **Warning:** This is a critical state transition hook. |
| setObjectiveData(objectiveUUID, chestUUID, itemStacks) | void | O(N) | A configuration method used during setup to link the chest to an objective and populate its inventory. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is uncommon. The primary integration pattern is to listen for the event it produces. Systems that manage quest progression subscribe to the TreasureChestOpeningEvent on the event bus.

```java
// Example of a listener in a separate system (e.g., ObjectivePlugin)
@Subscribe
public void onTreasureChestOpened(TreasureChestOpeningEvent event) {
    Objective targetObjective = objectiveStore.getObjective(event.getObjectiveUUID());
    if (targetObjective != null) {
        targetObjective.updateProgressForPlayer(event.getPlayerRef());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new TreasureChestState()` during gameplay. The world engine manages its lifecycle through serialization via its CODEC. Manual creation bypasses persistence and world state management.
-   **Bypassing State Transitions:** Do not call `setOpened(true)` directly. This circumvents the `onOpen` method, preventing the critical TreasureChestOpeningEvent from being dispatched. Systems listening for this event will fail to update, leading to desynchronized game state.
-   **Ignoring Validators:** Forcing an open or destroy action without first calling `canOpen` or `canDestroy` violates the core game logic this class is designed to enforce.

## Data Pipeline
The TreasureChestState acts as a validation and event-generation step in the player interaction pipeline.

> Flow:
> Player Interaction (Input) -> Server World Tick -> `World::tryOpenContainer` -> **TreasureChestState::canOpen** -> Validation Pass -> **TreasureChestState::onOpen** -> `HytaleServer::getEventBus` -> `TreasureChestOpeningEvent` -> Objective System (Listener) -> Player Quest Progress Update

