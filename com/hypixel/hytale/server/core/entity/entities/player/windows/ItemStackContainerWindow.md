---
description: Architectural reference for ItemStackContainerWindow
---

# ItemStackContainerWindow

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Transient

## Definition
```java
// Signature
public class ItemStackContainerWindow extends Window implements ItemContainerWindow {
```

## Architecture & Concepts
The ItemStackContainerWindow is a server-side representation of a UI window that is directly bound to a specific, single item stack which itself acts as a container (for example, a backpack item). It serves as a specialized view-controller, bridging the server's core inventory system with the player's client-side interface.

The key architectural pattern employed here is **reactive lifecycle management**. This window does not passively wait to be told it is invalid. Instead, upon opening, it actively subscribes to change events from its backing item's parent container. If an event occurs that invalidates the underlying item stack (e.g., the item is moved, dropped, or destroyed), the window automatically triggers its own closure. This self-destruct mechanism is critical for preventing state desynchronization between the server and the client, ensuring players cannot interact with a container that no longer exists in its original context.

This class is a concrete implementation of the abstract Window, specializing its behavior for item-based containers.

### Lifecycle & Ownership
- **Creation:** An instance is created by the server's window management system when a player performs an action to open a container-like item. The `ItemStackItemContainer` that this window will represent must be provided during construction.
- **Scope:** The object's lifetime is strictly coupled to the validity of the `ItemStack` it represents. It persists only while the player has the window open and the backing item remains in a valid state within its parent container.
- **Destruction:** Destruction is handled via two primary paths:
    1.  **Explicit Closure:** A client-side action (closing the UI) sends a packet to the server, which results in the `close` method being called on this instance.
    2.  **Implicit Closure:** The internal event listener detects that the backing `ItemStack` is no longer valid and programmatically calls the `close` method on itself. This is the most common and critical destruction path.

Upon closure, the `onClose0` method is invoked, which unregisters the event listener to prevent memory leaks.

## Internal State & Concurrency
- **State:** This class is stateful. Its primary state consists of:
    - A final reference to the `ItemStackItemContainer` it manages.
    - A nullable `EventRegistration` handle, which tracks its subscription to the inventory system's event bus. This handle is the core of its reactive lifecycle.
    - An empty `JsonObject` named windowData, reserved for future client-side properties.

- **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. All interactions, including its creation, lifecycle method invocations (`onOpen0`, `onClose0`), and event callbacks, are designed to execute exclusively on the main server game thread. Any off-thread access will lead to severe concurrency issues, including race conditions with the event bus and illegal state modifications.

## API Surface
The public API is minimal, primarily exposing the underlying data model. The core logic is handled by protected lifecycle methods inherited from the parent `Window` class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getData() | JsonObject | O(1) | Returns a JsonObject containing metadata for the client. Currently unused. |
| getItemContainer() | ItemContainer | O(1) | Provides direct access to the underlying item container data model. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by most game logic. It is managed by a higher-level player window or inventory manager. The manager is responsible for creating the window and initiating its lifecycle.

```java
// Pseudo-code demonstrating manager-level usage
ItemStackItemContainer backpackContainer = player.getInventory().getBackpackContainer();
ItemStackContainerWindow window = new ItemStackContainerWindow(backpackContainer);

// The manager opens the window for the player, which internally calls onOpen0()
player.getWindowManager().openWindow(window);
```

### Anti-Patterns (Do NOT do this)
- **Direct Lifecycle Invocation:** Do not call `onOpen0` or `onClose0` directly. These are internal hooks managed by the base `Window` class's `open` and `close` methods. Bypassing the main methods will break lifecycle state.
- **State Tampering:** Do not attempt to manually set the `eventRegistration` field to null. This will detach the window from its self-destruction mechanism, creating a ghost window that can lead to server errors and memory leaks.
- **Reusing Instances:** A closed window instance must not be reopened. Its event listeners have been unregistered, and it is considered defunct. A new instance must be created for a new interaction.

## Data Pipeline

The primary flow for this component is not a data pipeline but a **control and event flow** that ensures state consistency.

> Flow:
> Player Interaction Packet -> Server Logic Creates **ItemStackContainerWindow** -> `Window.open()` calls `onOpen0()` -> Event Listener is registered on Parent Container -> Parent Container is modified (e.g., item moved) -> Change Event is fired -> **ItemStackContainerWindow**'s listener executes -> `isItemStackValid()` returns false -> `this.close()` is called -> `onClose0()` unregisters listener -> Window is destroyed.

