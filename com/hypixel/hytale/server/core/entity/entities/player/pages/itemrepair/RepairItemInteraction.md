---
description: Architectural reference for RepairItemInteraction
---

# RepairItemInteraction

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages.itemrepair
**Type:** Transient

## Definition
```java
// Signature
public class RepairItemInteraction extends ChoiceInteraction {
```

## Architecture & Concepts
The RepairItemInteraction class is a server-side **Command Object** that encapsulates the complete logic for a player repairing a single item. It is a direct implementation of the Command design pattern, designed to be executed in response to a specific player action originating from a user interface.

This class acts as a transactional controller for complex inventory modifications. Its primary architectural role is to isolate the business logic of item repair from the UI and network layers. When a player confirms a repair action, an instance of this class is executed. It is responsible for:
1.  Calculating the repair outcome, including durability penalties.
2.  Atomically removing the repair materials from the player's inventory.
3.  Atomically replacing the damaged item with its repaired version.
4.  Providing clear feedback (chat messages, sound effects) to the player.
5.  Closing the associated UI page.

It operates directly on the core server data structures, including the Entity Component System (via Store and Ref) and the player's inventory containers. Its design ensures that if any step of the repair process fails, the transaction is safely rolled back to prevent item duplication or loss.

## Lifecycle & Ownership
-   **Creation:** An instance is created by the server-side logic that manages a specific UI page, such as an anvil or repair station. It is instantiated with the precise context of the item to be repaired, the item being consumed as material, and the associated repair penalty. This object is then attached as a potential choice to the player's active UI page.

-   **Scope:** Extremely short-lived. The object's lifetime is bound to a single user interaction. It is created, its *run* method is invoked exactly once, and it is then immediately eligible for garbage collection.

-   **Destruction:** The object is not managed and holds no persistent state. It is garbage collected after the `run` method completes. **WARNING:** Holding a reference to this object after execution is a memory leak and a design violation.

## Internal State & Concurrency
-   **State:** The internal state of a RepairItemInteraction instance is **effectively immutable**. The fields `itemContext`, `repairPenalty`, and `heldItemContext` are final and are set only during construction. The object serves as a data carrier for the `run` method and does not mutate its own state during execution.

-   **Thread Safety:** This class is **NOT thread-safe**. While the object's own state is immutable, its `run` method performs direct, unsynchronized mutations on shared server state, specifically the player's inventory and entity components within the `EntityStore`.

    **WARNING:** The `run` method **MUST** be executed exclusively on the main server thread that owns the target player's entity data. Invoking it from any other thread will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is defined by its constructor and the `run` method inherited from ChoiceInteraction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RepairItemInteraction(itemContext, repairPenalty, heldItemContext) | constructor | O(1) | Constructs a new command object with the required context for a repair operation. |
| run(store, ref, playerRef) | void | O(1) | Executes the transactional repair logic. Throws various exceptions on invalid state. |

## Integration Patterns

### Standard Usage
This object is not meant to be invoked arbitrarily. It is created and registered as a callback for a UI element, typically managed by the PageManager system. The system invokes the `run` method when the server receives a network packet indicating the player has selected the corresponding UI choice.

```java
// In a hypothetical AnvilBlockEntity class
// This code would run when a player opens the anvil UI

// 1. Define the context for the repair
ItemContext itemToRepair = ... // Get item from anvil input slot
ItemContext repairMaterial = ... // Get material from anvil material slot
double penalty = calculatePenalty();

// 2. Create the interaction command
RepairItemInteraction repairAction = new RepairItemInteraction(
    itemToRepair,
    penalty,
    repairMaterial
);

// 3. Add the command as a choice to the UI page
// The PageManager will later invoke repairAction.run() upon player confirmation.
player.getPageManager().addChoice("repair_button", repairAction);
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not hold a reference to this object and attempt to call `run` multiple times. It is a single-use command. Re-instantiate it for every new repair operation.
-   **Incorrect Context:** Do not create a RepairItemInteraction with stale or invalid ItemContext references. The context objects must point to the actual items in the player's inventory at the moment of execution.
-   **Execution Outside Lifecycle:** Do not instantiate and call `run` from a system that is not part of the player-UI interaction flow. Doing so bypasses the intended transactional and feedback mechanisms.

## Data Pipeline
The execution of this class is the final step in a client-to-server user interaction flow.

> Flow:
> Player clicks "Repair" in UI -> Client sends `PlayerChoicePacket` -> Server Network Layer decodes packet -> `PageManager` retrieves the registered **RepairItemInteraction** -> `PageManager` invokes `run()` method -> **RepairItemInteraction** modifies `EntityStore` (Inventory) -> **RepairItemInteraction** sends `PlayerMessagePacket` and `SoundEffectPacket` to client.

