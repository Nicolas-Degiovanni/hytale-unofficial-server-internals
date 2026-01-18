---
description: Architectural reference for DiagramCraftingWindow
---

# DiagramCraftingWindow

**Package:** com.hypixel.hytale.builtin.crafting.window
**Type:** Session-Scoped State Object

## Definition
```java
// Signature
public class DiagramCraftingWindow extends CraftingWindow implements ItemContainerWindow {
```

## Architecture & Concepts
The DiagramCraftingWindow is a server-side component that manages the state and logic for a "diagram" or "blueprint" based crafting station. Unlike a freeform grid, this window requires players to place a primary item to unlock specific recipe slots, which are then filled with secondary ingredients.

This class acts as the stateful controller for the crafting UI presented to a single player. It is responsible for:
-   Managing the lifecycle of the crafting session from open to close.
-   Handling player interactions sent via network packets (WindowAction).
-   Validating recipes against the items placed in its managed inventory containers.
-   Coordinating with the global CraftingManager to queue and execute crafting tasks.
-   Responding to changes in the player's main inventory to update UI hints.
-   Synchronizing its state with the client by marking itself as dirty (invalidate).

It is a critical bridge between the player's network session, their entity state (inventory), and the game's core crafting systems.

### Lifecycle & Ownership
-   **Creation:** An instance is created by the server's window management system when a player entity interacts with a block configured as a DiagramCraftingBench. It is not created directly.
-   **Scope:** The object's lifetime is strictly bound to the player's interaction with the crafting bench. It persists only as long as the corresponding UI is open on the client.
-   **Destruction:** The instance is marked for garbage collection when the player closes the window or moves too far from the bench. The onClose0 method is invoked as the primary cleanup hook, ensuring all items are returned to the player's inventory and all event listeners are unregistered.

**WARNING:** Failure to call onClose0 will result in a memory leak. The inventoryRegistration event listener will remain active, causing unnecessary processing and preventing the window object from being garbage collected. The presence of a check in the finalize method underscores the criticality of this cleanup path.

## Internal State & Concurrency
-   **State:** This class is highly mutable and stateful. It maintains a complex state graph including:
    -   The currently selected category and itemCategory strings.
    -   Multiple SimpleItemContainer and CombinedItemContainer objects representing the primary input slot, secondary ingredient slots, and the output slot.
    -   An EventRegistration handle for the player's inventory.
    -   Cached configuration data from the associated CraftingBench asset.

-   **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. It is designed to be accessed and modified exclusively by the server thread responsible for the associated player entity. All interactions are processed serially through the handleAction method or internal event callbacks. Unsynchronized access from other threads will lead to state corruption, inventory duplication, and server instability.

## API Surface
The public API is primarily composed of lifecycle methods invoked by the server's window management system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onOpen0() | boolean | O(N) | Initializes the window state, creates item containers, and registers an event listener on the player's inventory. Complexity is tied to recipe lookups for UI hint generation. |
| onClose0() | void | O(M) | Performs critical cleanup. Returns all items from the crafting grid to the player's inventory and unregisters the inventory event listener. M is the number of item stacks in the input containers. |
| handleAction(ref, store, action) | void | O(R) | The primary entry point for player input. Dispatches logic based on the WindowAction type. Complexity is highest for CraftItemAction, which requires recipe validation (R). |
| getItemContainer() | ItemContainer | O(1) | Provides access to the combined view of all item containers managed by this window, allowing external systems to inspect its inventory state. |

## Integration Patterns

### Standard Usage
Direct interaction is uncommon. Systems typically interact with this window by sending network actions to the player who has it open. This decouples the calling system from the window's internal implementation.

```java
// Example: A quest system programmatically triggering a craft action
// for the player associated with 'playerRef'.

// This packet is constructed and sent to the player's client connection.
// The server's network layer routes it to the player's active window.
CraftItemAction action = new CraftItemAction();
playerRef.getComponent(Player.class).sendPacket(action);

// The DiagramCraftingWindow instance receives this via its handleAction method.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new DiagramCraftingWindow()`. The window system manages its lifecycle. Manual creation will result in a non-functional object that is not linked to any player.
-   **State Tampering:** Do not directly get and modify the internal ItemContainer objects (e.g., inputPrimaryContainer). This bypasses validation, event handling (updateInput), and state synchronization logic, leading to a desync between the server state and the client UI.
-   **Leaked Listeners:** Forgetting to call onClose0 is a critical error. The inventoryRegistration will remain active, causing a memory leak and performance degradation as the orphaned window continues to react to inventory events.

## Data Pipeline
The flow of data for a successful craft operation demonstrates the component's role as a state coordinator.

> Flow:
> Client UI Event (Player clicks "Craft") -> **CraftItemAction Packet** -> Server Network Layer -> Player's Active Window -> **DiagramCraftingWindow.handleAction** -> Internal state validation and recipe lookup (`collectRecipes`) -> **CraftingManager.queueCraft** -> Timed craft processing -> Inventory transaction (removes ingredients, adds result) -> **DiagramCraftingWindow.updateInput** (triggered by inventory change event) -> `invalidate()` call -> Window Update Packet -> Client UI Update

