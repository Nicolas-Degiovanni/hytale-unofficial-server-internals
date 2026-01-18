---
description: Architectural reference for FieldCraftingWindow
---

# FieldCraftingWindow

**Package:** com.hypixel.hytale.builtin.crafting.window
**Type:** Transient

## Definition
```java
// Signature
public class FieldCraftingWindow extends Window {
```

## Architecture & Concepts

The FieldCraftingWindow is the server-side state model and controller for the player's inventory-accessible crafting interface, often called "pocket crafting" or "field crafting". It is a specialized implementation of the abstract Window class, designed to manage the UI state and handle user interactions for crafting that does not require a physical crafting bench.

Architecturally, this class serves as a crucial link between static game data, dynamic player state, and the network protocol. Its primary responsibility is to assemble a comprehensive JSON data model that the client can render. This model is not static; it is built by aggregating data from multiple sources:

1.  **Static Asset Data:** The fundamental structure, including categories, names, and icons, is loaded directly from FieldcraftCategory game assets. This makes the UI highly configurable by designers without requiring code changes.
2.  **Plugin-Managed Recipe Data:** It queries the CraftingPlugin to determine the specific set of recipes available to the player within each category. This decouples the window's presentation logic from the rules governing recipe availability.
3.  **Dynamic World State:** Upon opening, it injects dynamic, session-specific data, such as the world's "Memories Level", which can influence the UI or available actions.

When a player interacts with the UI (e.g., clicks a "craft" button), the client sends a WindowAction packet. The FieldCraftingWindow's handleAction method acts as the server-side endpoint for this interaction, delegating the core crafting logic to the appropriate systems like the CraftingManager.

### Lifecycle & Ownership

-   **Creation:** An instance of FieldCraftingWindow is created by the server's window management system when a player initiates an action that opens the pocket crafting screen. It is never instantiated directly by gameplay logic.
-   **Scope:** The object's lifetime is tied directly to the player's UI session. It persists only as long as the player has the field crafting window open.
-   **Destruction:** The instance is marked for garbage collection as soon as the player closes the window or disconnects from the server. The empty onClose0 method indicates that no explicit resource cleanup is required.

## Internal State & Concurrency

-   **State:** This class is stateful and highly mutable during its initialization phase. It maintains a single JsonObject, windowData, which caches the complete UI model. This model is constructed once and then updated with dynamic data when the window is opened via onOpen0.
-   **Thread Safety:** **This class is not thread-safe.** All methods are expected to be invoked exclusively on the main world tick thread. The underlying components it interacts with, such as the EntityStore and CraftingManager, are not designed for concurrent access. Any attempt to call its methods from an asynchronous task or different thread will result in severe state corruption and server instability.

## API Surface

The public contract is primarily defined by its lifecycle methods, which are invoked by the parent windowing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getData() | JsonObject | O(1) | Returns a reference to the cached JSON model representing the window's state. |
| onOpen0() | boolean | O(1) | Lifecycle hook. Populates the data model with dynamic world state before it is sent to the client. |
| onClose0() | void | O(1) | Lifecycle hook. Invoked when the window is closed. Currently performs no action. |
| handleAction(ref, store, action) | void | O(1) | Handles incoming user actions from the client, delegating crafting logic to the CraftingManager. |

## Integration Patterns

### Standard Usage

Developers should never interact with this class directly. The engine's window management system is solely responsible for its lifecycle. Gameplay systems can influence the window indirectly by modifying the underlying data sources it reads from, such as registering new recipes with the CraftingPlugin.

The primary interaction flow is managed by the engine:

1.  Player presses the keybind for pocket crafting.
2.  Server's WindowManager instantiates FieldCraftingWindow for that player.
3.  The WindowManager calls onOpen0.
4.  The window's data is serialized and sent to the client.
5.  Player clicks to craft an item, sending a CraftRecipeAction packet.
6.  The network layer routes the packet to the FieldCraftingWindow instance's handleAction method.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call new FieldCraftingWindow(). The window system manages its creation and association with a player. Direct creation will result in a disconnected, non-functional object.
-   **External State Mutation:** Do not attempt to get an instance of this window and call getData() to modify the JsonObject from another system. This breaks the component's ownership model and can lead to unpredictable UI behavior. State changes should be driven by modifying the source data (assets, plugins) that the window consumes during its creation.
-   **Asynchronous Handling:** Do not call handleAction from any thread other than the main world thread. All player-related actions must be synchronized with the game tick.

## Data Pipeline

The flow of data from configuration to player interaction is a multi-stage process, demonstrating a clear separation of concerns between static data, runtime logic, and player input.

> **Flow (Initialization & Display):**
> Game Assets (FieldcraftCategory) -> CraftingPlugin (Recipe Registry) -> **FieldCraftingWindow Constructor** -> MemoriesPlugin -> **onOpen0** -> Serializer -> Network Packet -> Client UI Render
>
> **Flow (Player Interaction):**
> Client UI Click -> Network Packet (CraftRecipeAction) -> Packet Decoder -> **handleAction** -> CraftingManager -> EntityStore (Player Inventory) -> SoundUtil (Audio Feedback)

