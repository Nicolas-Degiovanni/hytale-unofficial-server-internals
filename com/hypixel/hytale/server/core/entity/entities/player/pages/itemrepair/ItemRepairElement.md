---
description: Architectural reference for ItemRepairElement
---

# ItemRepairElement

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages.itemrepair
**Type:** Transient

## Definition
```java
// Signature
public class ItemRepairElement extends ChoiceElement {
```

## Architecture & Concepts
The ItemRepairElement class is a server-side **View Model** responsible for representing a single repairable item within a player's user interface. It is not a UI component itself, but rather a data structure that drives the generation of UI commands sent to the client.

This class acts as a specialized implementation of the more generic ChoiceElement, tailoring the UI generation logic specifically for displaying an ItemStack's properties, such as its icon, name, and current durability. It forms a critical link in the server-authoritative UI system, where the server describes UI state changes via a command buffer, and the client simply executes these commands to render the final view. This pattern ensures that all UI-related game logic remains on the server.

ItemRepairElement encapsulates the translation of a game state object, the ItemStack, into a set of specific, render-agnostic UI instructions.

### Lifecycle & Ownership
- **Creation:** An ItemRepairElement is instantiated on-demand by a higher-level UI page controller, typically when a player interacts with a game world object that triggers an item repair menu (e.g., an anvil). A new instance is created for each repairable item that needs to be displayed.
- **Scope:** The object's lifetime is exceptionally short. It exists only for the duration of the UI generation process for a single screen. It is created, its addButton method is called to populate a UICommandBuilder, and it is then immediately discarded.
- **Destruction:** The object becomes eligible for garbage collection as soon as the parent UI page has finished building its command list. It holds no persistent state and is not referenced outside of the immediate UI construction scope.

## Internal State & Concurrency
- **State:** The class holds a reference to an ItemStack. While the ItemRepairElement itself does not modify this state, the underlying ItemStack is a mutable object. However, in the context of its lifecycle, the ItemRepairElement effectively acts as an immutable snapshot of the item's state at the moment of UI creation.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, used, and discarded exclusively on the main server thread within a single game tick. Accessing it from any other thread is an unsupported operation and will lead to race conditions with the game state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ItemRepairElement(itemStack, interaction) | constructor | O(1) | Constructs a new element for a given ItemStack and its associated repair interaction. |
| addButton(commandBuilder, ...) | void | O(1) | Populates the provided UICommandBuilder with the necessary commands to render this item element on the client. |

## Integration Patterns

### Standard Usage
This class is intended to be used by a parent UI "Page" or "Container" object. The typical pattern involves iterating through a collection of eligible items, creating an ItemRepairElement for each, and using it to populate a list in the UI.

```java
// Example from a hypothetical parent UI builder
UICommandBuilder commandBuilder = new UICommandBuilder();
List<ItemStack> repairableItems = player.getRepairableItems();

for (ItemStack item : repairableItems) {
    // Create a specific interaction for this item
    RepairItemInteraction interaction = new RepairItemInteraction(item);

    // Create the view model for this item
    ItemRepairElement element = new ItemRepairElement(item, interaction);

    // Use the element to generate UI commands for a list entry
    element.addButton(commandBuilder, eventBuilder, "#RepairList.Add", playerRef);
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not cache and reuse ItemRepairElement instances across multiple UI updates or for different players. The underlying ItemStack state may change between ticks, and reusing an element will result in displaying stale data.
- **Modification After Construction:** Do not attempt to modify the internal ItemStack after passing it to the constructor. The UI generation logic assumes the state is fixed at the time of creation.
- **Use Outside UI Generation:** This class has no purpose outside of populating a UICommandBuilder. Using it for general game logic is a misuse of its design.

## Data Pipeline
The ItemRepairElement functions as a transformation step in the server-to-client UI data flow.

> Flow:
> Player Interaction (e.g., uses anvil) -> Server Game Logic (fetches repairable items) -> **ItemRepairElement Instantiation** -> `addButton` Method Translates ItemStack to UI Commands -> UICommandBuilder -> Network Packet -> Client UI Engine -> Rendered Item in List

