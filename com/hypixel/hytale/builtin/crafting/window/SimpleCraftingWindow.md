---
description: Architectural reference for SimpleCraftingWindow
---

# SimpleCraftingWindow

**Package:** com.hypixel.hytale.builtin.crafting.window
**Type:** Stateful Object

## Definition
```java
// Signature
public class SimpleCraftingWindow extends CraftingWindow implements MaterialContainerWindow {
```

## Architecture & Concepts
The SimpleCraftingWindow is a server-side logical container that represents a player's active session with a basic, non-tiered crafting bench. It is not the user interface itself, but rather the state machine and action handler that drives the UI's behavior from the server.

This class acts as a crucial bridge between the network protocol layer and the core game logic systems. When a player performs an action in the crafting UI (e.g., clicking the "craft" button), the client sends a `WindowAction` packet. The server's `WindowManager` routes this action to the corresponding SimpleCraftingWindow instance, which then orchestrates the necessary changes to the world state.

Its primary responsibilities include:
1.  **Action Dispatching:** Receiving generic `WindowAction` objects and delegating them to the appropriate subsystems, primarily the `CraftingManager`.
2.  **State Validation:** Ensuring that crafting attempts are valid by checking recipe existence and player inventory.
3.  **System Orchestration:** Coordinating interactions between the `CraftingManager`, the player's `Inventory`, and the `World` for effects like sound playback.
4.  **State Feedback:** Updating the state of the associated world block (the crafting bench) to reflect ongoing processes like timed crafts or upgrades.

It operates entirely within the server's Entity-Component-System (ECS) framework, identified by the `Ref<EntityStore>` and `Store<EntityStore>` parameters in its core methods. This design ensures all state modifications are managed through the central game loop.

### Lifecycle & Ownership
-   **Creation:** A SimpleCraftingWindow is instantiated by the server's `WindowManager` when a player successfully interacts with a compatible crafting bench block. The state of that specific block, encapsulated in a `BenchState` object, is injected upon construction. It is never created directly.
-   **Scope:** The object's lifetime is strictly tied to the player's UI session. It persists only as long as the player has the crafting window open and remains in proximity to the bench.
-   **Destruction:** The instance is marked for garbage collection by the `WindowManager` when the player closes the window, moves too far away from the associated block, or disconnects from the server.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It holds a reference to the `BenchState` of the physical crafting bench in the world. Through its implementation of `MaterialContainerWindow`, it also manages an internal item container for supplementary crafting ingredients provided by the bench itself. Its state is a direct reflection of a live player interaction.

-   **Thread Safety:** **This class is not thread-safe and must not be accessed from multiple threads.** All interactions with a SimpleCraftingWindow instance are expected to occur on the main server thread that ticks the `World` and its associated `EntityStore`. The ECS architecture inherently serializes access, preventing data corruption. Any attempt to call its methods from an asynchronous task or worker thread will lead to severe state desynchronization and server instability.

## API Surface
The public API is minimal, exposing only the necessary lifecycle and action handling methods to the managing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(playerRef, manager) | void | O(1) | Second-stage constructor. Links the window to a specific player and the `WindowManager`. **WARNING:** The object is not in a valid state until this method is called. |
| handleAction(ref, store, action) | void | O(N) | The primary entry point for all logic. Processes a `WindowAction` from the client. Complexity depends on the crafting operation. Throws exceptions on invalid state. |

## Integration Patterns

### Standard Usage
A SimpleCraftingWindow is exclusively managed by the `WindowManager`. A developer will never interact with this class directly. The system's internal flow processes a network packet and routes it to the correct window instance associated with the player.

```java
// This code is conceptual and represents how the WindowManager would use this class.
// A developer would NOT write this.

// 1. A CraftRecipeAction packet arrives for a player.
WindowAction action = receivedPacket.getAction();
PlayerRef playerRef = getPlayerRefForConnection(connection);
WindowManager windowManager = playerRef.getWindowManager();

// 2. The manager retrieves the player's currently open window.
SimpleCraftingWindow currentWindow = (SimpleCraftingWindow) windowManager.getOpenWindow();

// 3. The action is dispatched to the window for processing.
if (currentWindow != null) {
    // The window handles all interaction with the CraftingManager and Inventory.
    currentWindow.handleAction(playerEntityRef, world.getEntityStore(), action);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SimpleCraftingWindow()`. The `WindowManager` is the sole owner and creator of window instances. Manual creation will result in a disconnected object that does not receive actions and cannot affect game state.
-   **External State Mutation:** Do not attempt to get the window's internal item container and modify it from another system. All inventory and state changes must be initiated through the `handleAction` method to ensure game rules are correctly applied.
-   **Asynchronous Access:** Calling `handleAction` from a separate thread to "pre-load" a craft or for any other reason is strictly forbidden. This will bypass the ECS framework's state management and corrupt the `EntityStore`.

## Data Pipeline
The flow of data for a crafting request is unidirectional, originating from the client and resulting in a server-side state change.

> Flow:
> Client UI Interaction -> `CraftRecipeAction` Packet -> Server Network Handler -> `WindowManager` -> **SimpleCraftingWindow.handleAction** -> `CraftingManager` -> Player Inventory & World State Mutation -> (Optional) Network Packet to Client for UI update

