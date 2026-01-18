---
description: Architectural reference for ContainerWindow
---

# ContainerWindow

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Transient

## Definition
```java
// Signature
public class ContainerWindow extends Window implements ItemContainerWindow {
```

## Architecture & Concepts
The ContainerWindow class is a server-side abstraction that represents a player's active view of an inventory. It serves as a critical adapter between a raw data model, the ItemContainer, and the player session management system, the Window framework.

Its primary architectural role is to bind a specific inventory instance (e.g., a chest's contents, a crafting grid) to a generic window concept that the server can manage and synchronize with the client. When a player interacts with an object that has an inventory, the server instantiates a ContainerWindow, effectively creating a temporary, stateful session for that interaction. All subsequent player actions on that inventory, suchas moving or crafting items, are routed through this window object to the underlying ItemContainer.

This class is a concrete implementation of the ItemContainerWindow interface, enforcing a contract that guarantees access to an ItemContainer.

### Lifecycle & Ownership
- **Creation:** A ContainerWindow is instantiated dynamically at runtime by server-side logic, typically within an interaction handler. For example, when a player entity interacts with a block entity that possesses an inventory, the system creates a new ContainerWindow, injecting the block's ItemContainer into the constructor. It is never created declaratively or pre-emptively.
- **Scope:** The object's lifetime is strictly bound to the duration of the player's interaction. It persists only as long as the corresponding UI window is considered open by the server for a specific player.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the window is closed. Closure can be initiated by the client (e.g., pressing the escape key) or by the server (e.g., the player moves too far from the associated container). The PlayerWindowManager is responsible for managing this lifecycle.

## Internal State & Concurrency
- **State:** ContainerWindow is stateful. It maintains two final fields:
    1.  **itemContainer:** A reference to the live, mutable inventory data model. The ContainerWindow does not own this data but acts as a gateway to it.
    2.  **windowData:** A JsonObject for arbitrary metadata. While the reference is final, the JsonObject itself is mutable.

- **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. All operations on a ContainerWindow instance, and especially on the ItemContainer it holds, must be performed exclusively on the main server thread (the game tick thread).

    **WARNING:** Accessing or modifying the underlying ItemContainer from an asynchronous task or a different thread will lead to severe concurrency issues, including data corruption, client-server desynchronization, and server crashes. All inventory mutations must be scheduled as tasks to be executed on the main game loop.

## API Surface
The public API provides access to the underlying data model and lifecycle hooks managed by the parent Window system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemContainer() | ItemContainer | O(1) | Returns the backing ItemContainer. This is the primary method for accessing and manipulating the inventory data. |
| getData() | JsonObject | O(1) | Returns a mutable JSON object used to store and transmit auxiliary window properties to the client. |
| onOpen0() | boolean | O(1) | Internal lifecycle hook invoked by the Window system when the window is opened. |
| onClose0() | void | O(1) | Internal lifecycle hook invoked by the Window system when the window is closed. |

## Integration Patterns

### Standard Usage
The standard pattern involves creating a ContainerWindow to grant a player temporary access to an inventory. This is typically done within a block or entity interaction handler.

```java
// Example: In a system that handles player interaction with a chest
ItemContainer chestInventory = chestBlockEntity.getInventory();
Player player = interactionEvent.getPlayer();

// Create a new window instance for this specific interaction
ContainerWindow window = new ContainerWindow(chestInventory);

// The PlayerWindowManager now takes ownership and sends the open packet
player.getWindowManager().openWindow(window);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not attempt to cache and re-use a ContainerWindow instance for multiple players or multiple interactions. Each time a container is opened, a new ContainerWindow must be instantiated to ensure a clean session state.
- **State Caching:** Do not retrieve the ItemContainer and store a long-lived reference to it elsewhere. The window may be closed at any moment, invalidating the context of the interaction. Always access the ItemContainer through the player's currently active window.
- **External Modification:** Do not modify the ItemContainer from a separate system while a player has a ContainerWindow open to it, unless that system is synchronized with the main server thread and is aware of the active UI session. This can cause client-side prediction errors and inventory desynchronization.

## Data Pipeline
The ContainerWindow acts as a server-side controller in the player inventory interaction pipeline.

> **Flow: Opening a Window**
> Player Input (Interact with Chest) → Server Interaction System → `new ContainerWindow(chestInventory)` → PlayerWindowManager → **ContainerWindow** → Network Layer (sends OpenWindow packet) → Client UI

> **Flow: Clicking an Item**
> Client UI Action (Click Slot) → Network Layer (sends WindowClick packet) → Server Packet Handler → PlayerWindowManager (finds active window) → **ContainerWindow**.getItemContainer() → ItemContainer.executeClick() → Server sends inventory updates

