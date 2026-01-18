---
description: Architectural reference for CraftingWindow
---

# CraftingWindow

**Package:** com.hypixel.hytale.builtin.crafting.window
**Type:** Transient State Object

## Definition
```java
// Signature
public abstract class CraftingWindow extends BenchWindow {
```

## Architecture & Concepts
The CraftingWindow class is the server-side representation of a player's active user interface for a crafting bench. It is a critical component that acts as a bridge between the player's client, the game world, and the core crafting systems.

Its primary architectural responsibilities are:

1.  **State Serialization:** Upon creation, it queries the game's asset and recipe database (`CraftingPlugin`, `CraftingBench` configuration) to assemble a comprehensive JSON payload. This payload, stored in the `windowData` field, contains all information the client needs to render the crafting UI, including categories, available recipes, and icons.
2.  **Lifecycle Management:** It manages the state of the physical crafting bench block in the world. It hooks into the open and close events of the UI to play sounds and, most importantly, to update the block's interaction state via `setBlockInteractionState` upon closing the window.
3.  **Action Processing:** Through the static `craftSimpleItem` method, it serves as the entry point for handling crafting requests sent from the client. It validates the request and delegates the complex logic of inventory manipulation and item creation to the `CraftingManager` service.

This class is not a long-lived service but a temporary, stateful object instantiated for the duration of a single player-bench interaction.

## Lifecycle & Ownership
-   **Creation:** A CraftingWindow instance is created by the server's window management system when a player successfully interacts with a crafting bench block in the world. The state of that specific block, encapsulated in a `BenchState` object, is provided during construction.
-   **Scope:** The object's lifetime is strictly tied to the player's UI session with the crafting bench. It persists only as long as the crafting window is open on the player's client.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection immediately after the `onClose0` method is invoked. This occurs when the player manually closes the window, moves too far from the bench, or disconnects from the server.

## Internal State & Concurrency
-   **State:** This object is highly stateful and mutable. Its primary state is the `windowData` JsonObject, which is assembled in the constructor and further modified during the `onOpen0` lifecycle hook. It also maintains a direct reference to the `BenchState` of the associated world block.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, accessed, and destroyed exclusively by the server thread responsible for processing the owning player's game ticks and network packets. Any access from other threads will result in severe concurrency issues, including world state corruption and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onOpen0() | boolean | O(N) | Hook called when the window is opened. Serializes recipe and gameplay data for the client. Complexity is proportional to the number of recipes for the bench. |
| onClose0() | void | O(1) | Hook called when the window is closed. Updates the world block's state and plays a sound. |
| setBlockInteractionState(state, world, settings) | void | O(1) | Directly mutates the state of the associated block in the world chunk. |
| craftSimpleItem(store, ref, manager, action) | static boolean | O(1) | Handles a `CraftRecipeAction` packet. Validates the recipe and delegates to the CraftingManager. **Warning:** Disconnects the player on an invalid recipe ID. |

## Integration Patterns

### Standard Usage
A developer does not typically instantiate or directly manage a CraftingWindow. The server's interaction and window systems handle its lifecycle. The primary integration point is the static `craftSimpleItem` method, which is invoked by the network packet handler for crafting.

```java
// Invoked within a network packet handler for CraftRecipeAction
public void handleCraftRecipe(CraftRecipeAction action) {
    // The system retrieves the player's context (Store, Ref) and the
    // global CraftingManager service.
    Store<EntityStore> playerStore = getPlayerStore();
    Ref<EntityStore> playerRef = getPlayerRef();
    CraftingManager craftingManager = getCraftingManager();

    // The static method is called to process the crafting request.
    boolean success = CraftingWindow.craftSimpleItem(
        playerStore,
        playerRef,
        craftingManager,
        action
    );

    if (!success) {
        // Log failed crafting attempt.
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new CraftingWindow()`. The server's window management framework is the sole owner of its creation and destruction. Manual instantiation will result in a disconnected object that does not affect the game state.
-   **State Caching:** Do not hold a reference to a CraftingWindow instance beyond the scope of a single interaction. These objects are ephemeral and will be invalid after the player closes the UI.
-   **External State Modification:** Do not modify the `windowData` JsonObject after the `onOpen0` method has completed. The data will have already been sent to the client, and further changes will have no effect.

## Data Pipeline
The flow of data and control for the two primary operations (opening the window and crafting) are distinct.

> **Window Opening Flow:**
> Player interacts with Block -> Server Window System -> **new CraftingWindow(benchState)** -> `onOpen0()` populates `windowData` -> `windowData` is serialized to network packet -> Client receives packet and renders UI

> **Item Crafting Flow:**
> Client sends `CraftRecipeAction` packet -> Server Network Handler -> **CraftingWindow.craftSimpleItem()** -> `CraftingManager.craftItem()` -> Player Inventory is mutated -> Inventory update packet sent to Client

