---
description: Architectural reference for StructuralCraftingWindow
---

# StructuralCraftingWindow

**Package:** com.hypixel.hytale.builtin.crafting.window
**Type:** Stateful Component

## Definition
```java
// Signature
public class StructuralCraftingWindow extends CraftingWindow implements ItemContainerWindow {
```

## Architecture & Concepts
The StructuralCraftingWindow is a server-side component that models the state and logic for a "structural" or single-input crafting bench, such as a sawmill or stonecutter. It acts as the stateful controller for the UI window a player interacts with.

This class serves as a critical bridge between the server's core systems and player actions received over the network. It translates low-level `WindowAction` packets (e.g., `SelectSlotAction`, `CraftRecipeAction`) into high-level game logic that interacts with the `CraftingManager` and the player's `Inventory`.

Internally, it is composed of two primary inventories:
1.  **inputContainer:** A single-slot container for the source material.
2.  **optionsContainer:** A 64-slot container that displays the possible crafting outputs derived from the input material.

These are wrapped in a `CombinedItemContainer` to present a unified inventory view to the client and other server systems. The window is responsible for dynamically calculating the list of available recipes by calling `updateRecipes` whenever the input item changes.

### Lifecycle & Ownership
-   **Creation:** An instance is created by the server's window management system when a player opens a compatible structural crafting bench in the world. The constructor requires a `BenchState` object, which provides the context of the specific bench being used.
-   **Scope:** The object's lifetime is strictly tied to the player's UI session. It persists only as long as the player has the corresponding crafting window open.
-   **Destruction:** The instance is marked for garbage collection after the `onClose0` method is invoked. This occurs when the player closes the window or is disconnected. This method is critical for cleanup, as it returns the input item to the player's inventory, cancels any queued crafting operations, and unregisters event listeners.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains the current input item, the list of calculated recipe options, the player's currently selected recipe slot (`selectedSlot`), and a mapping from UI slots to recipe identifiers (`optionSlotToRecipeMap`). Its state is a direct reflection of the player's UI.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be accessed exclusively from the main server game loop thread. All interactions, including handling network actions and inventory events, must be synchronized with the server tick to prevent race conditions and data corruption.

## API Surface
The public API is primarily composed of lifecycle hooks and an action handler, designed to be called by the server's windowing framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handleAction(ref, store, action) | void | O(N) | The primary entry point for processing player input. Dispatches logic based on the `WindowAction` type. Complexity varies; recipe lookups can be O(N) on the number of recipes for the bench. |
| getItemContainer() | ItemContainer | O(1) | Returns the combined view of the input and options containers. |
| onOpen0() | boolean | O(N) | Lifecycle hook called when the window is opened. Sets up initial state and registers listeners on the player's inventory. |
| onClose0() | void | O(1) | Lifecycle hook called when the window is closed. Performs essential cleanup, returning items and unregistering listeners. |

## Integration Patterns

### Standard Usage
A developer does not instantiate this class directly. The server framework creates it and manages its lifecycle. The primary interaction pattern is for the framework to delegate `WindowAction` packets to the `handleAction` method.

```java
// System-level code (Illustrative)
// A player action packet arrives for an open StructuralCraftingWindow instance.
WindowAction action = receivePacket();
StructuralCraftingWindow window = player.getActiveWindow();

// The framework dispatches the action to the window instance.
window.handleAction(playerRef, entityStore, action);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new StructuralCraftingWindow()`. The window requires a `BenchState` and must be managed by the server's UI framework to be correctly associated with a player session.
-   **External State Mutation:** Do not directly modify its internal containers (e.g., `window.inputContainer.clear()`). This will bypass the `updateRecipes` logic and lead to a desynchronized state between the server model and the client's view. All state changes must originate from the `handleAction` method.
-   **Asynchronous Access:** Accessing this object from a separate thread will lead to severe concurrency issues. All method calls must be performed on the main server thread.

## Data Pipeline
The flow of data for a typical crafting operation demonstrates the role of this class as a controller.

> Flow:
> Client Input (Click Craft) -> Network Packet (`CraftRecipeAction`) -> Server Window Manager -> **StructuralCraftingWindow.handleAction** -> `CraftingManager.queueCraft` -> Player Inventory Update -> **StructuralCraftingWindow.invalidate** -> Network Packet (Window Data Update) -> Client UI Refresh

