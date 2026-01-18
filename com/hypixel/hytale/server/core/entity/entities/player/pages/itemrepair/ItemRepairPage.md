---
description: Architectural reference for ItemRepairPage
---

# ItemRepairPage

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages.itemrepair
**Type:** Transient

## Definition
```java
// Signature
public class ItemRepairPage extends ChoiceBasePage {
```

## Architecture & Concepts
The ItemRepairPage class is a server-side UI Model that represents the item repair interface presented to a player. It is not a user interface component itself, but rather the data and logic source that drives the client-side UI. It functions as a specialized implementation of ChoiceBasePage, designed specifically to present a list of repairable items from a given inventory.

Its core architectural purpose is to act as a translator between the server's abstract inventory system and the concrete UI commands sent to the client. It encapsulates the business logic for determining which items are eligible for repair by scanning an ItemContainer, filtering for damaged and non-unbreakable items, and wrapping them in ItemRepairElement view models.

This class is a critical component in the player interaction loop for world objects like anvils or repair stations. It creates a temporary, context-specific state for the duration of the player's interaction.

## Lifecycle & Ownership
- **Creation:** An instance of ItemRepairPage is created on-demand by a higher-level game logic system, typically in response to a player interacting with a specific block or entity. The creator is responsible for supplying the full context: the player reference, the inventory to scan, the repair cost penalty, and the item context being used for the repair.

- **Scope:** The object's lifetime is ephemeral and is strictly bound to the time the player has the repair UI open. It is a short-lived object representing a single, atomic UI interaction.

- **Destruction:** The object is eligible for garbage collection as soon as the player navigates away from the repair screen. The Player's PageManager releases its reference, and since no other system retains a handle to it, it is cleaned up by the JVM. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** ItemRepairPage is stateful, but its state is effectively immutable after construction. The list of repairable items is generated once by the static getItemElements method, and this list is passed to the superclass constructor. The internal state, representing the choices available to the player, does not change for the lifetime of the object.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, operated on, and discarded entirely within the scope of a single server tick for a single player. All interactions with this class and its dependencies, such as UICommandBuilder, must be confined to the primary server thread responsible for the player's game state.

## API Surface
The public API is minimal, primarily consisting of the constructor and the overridden build method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ItemRepairPage(playerRef, itemContainer, repairPenalty, heldItemContext) | constructor | O(N) | Creates a new page instance. Complexity is linear based on N, the capacity of the itemContainer, due to the full inventory scan. |
| build(ref, commandBuilder, eventBuilder, store) | void | O(M) | Populates the UI builders with commands to render the page. Complexity is linear based on M, the number of repairable items found. |

## Integration Patterns

### Standard Usage
This class should be instantiated and immediately passed to a player's PageManager to be displayed. The logic typically resides within an interaction handler for a block or entity.

```java
// Example from a hypothetical Anvil block interaction handler
void onPlayerInteract(PlayerRef playerRef, ItemContainer inventory) {
    // Define repair parameters based on game rules
    double repairPenalty = 0.15;
    ItemContext heldItemContext = playerRef.getHeldItemContext();

    // Create the page with the player's current inventory state
    ItemRepairPage repairPage = new ItemRepairPage(playerRef, inventory, repairPenalty, heldItemContext);

    // Open the page for the player
    playerRef.getPageManager().open(repairPage);
}
```

### Anti-Patterns (Do NOT do this)
- **Caching or Re-use:** Do not store and re-use an ItemRepairPage instance. A new instance **must** be created each time the UI is opened to ensure it reflects the most current state of the player's inventory.
- **State Mutation:** Do not attempt to modify the page's elements after it has been constructed. The design relies on the list of choices being immutable for the object's lifetime.
- **Off-thread Access:** Never create or interact with this class from a worker thread. All operations must be on the main server thread to prevent race conditions with the player's state and the UI command pipeline.

## Data Pipeline
The ItemRepairPage is a key processing step in the flow of data from a player action to a rendered user interface.

> Flow:
> Player Interaction (e.g., Right-clicks Anvil) -> Server-Side Interaction Handler -> **ItemRepairPage Instantiation & Inventory Scan** -> PageManager opens the page -> `build()` method populates UICommandBuilder -> Serialized UI Commands -> Network Packet -> Client UI Engine -> Rendered Repair Screen

