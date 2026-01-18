---
description: Architectural reference for ContainerBlockWindow
---

# ContainerBlockWindow

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Transient

## Definition
```java
// Signature
public class ContainerBlockWindow extends BlockWindow implements ItemContainerWindow {
```

## Architecture & Concepts
The ContainerBlockWindow is a server-side state machine that represents a player's active interaction with an in-world block that contains items, such as a chest or a furnace. It serves as a critical bridge between the network protocol layer and the server's core inventory system.

Architecturally, this class follows a **Composition over Inheritance** pattern. While it inherits from BlockWindow to gain basic properties of a window tied to a world location, its primary functionality comes from its composition with an **ItemContainer**. The ContainerBlockWindow does not manage item state itself; it delegates all inventory operations—such as sorting, adding, or removing items—to the injected ItemContainer instance.

This design decouples the UI interaction logic (handling packets like SortItemsAction) from the data model (the ItemContainer), allowing the same inventory data to be represented by different windows or accessed by other game systems directly.

## Lifecycle & Ownership
- **Creation:** A ContainerBlockWindow is instantiated by a higher-level manager, typically a WindowManager or PlayerInteractionManager, when a player successfully opens a block entity that has an associated ItemContainer. The ItemContainer, which represents the block's persistent inventory state, is retrieved from the world's BlockEntity and passed into the ContainerBlockWindow's constructor.

- **Scope:** The object's lifetime is strictly tied to the player's UI session. It exists only for the duration that the player has the container's interface open.

- **Destruction:** The instance is marked for garbage collection as soon as the player closes the window, either through direct action (pressing an escape key) or indirect action (moving too far from the block). The managing system is responsible for clearing all references to the instance upon closure.

## Internal State & Concurrency
- **State:** This class is highly stateful, but its state is primarily managed by proxy. The internal ItemContainer field is a mutable reference to the block's actual inventory. Actions performed on the window, such as sorting, directly mutate the state of the ItemContainer. The JsonObject field, windowData, is effectively immutable after construction.

- **Thread Safety:** **This class is not thread-safe and must not be considered as such.** It is designed to be accessed and manipulated exclusively by the single main server thread responsible for the world tick. All interactions, including packet handling via handleAction, must be synchronized with the game loop to prevent race conditions and data corruption within the underlying ItemContainer and EntityStore.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getData() | JsonObject | O(1) | Returns static metadata used to construct the client-side UI, such as the block's item ID. |
| getItemContainer() | ItemContainer | O(1) | Provides direct access to the underlying inventory model for this window. |
| handleAction(ref, store, action) | void | O(N log N) | Processes a network action from the client. Complexity is dominated by the potential sorting operation. |

## Integration Patterns

### Standard Usage
The class is intended to be created and managed by a central windowing system. The ItemContainer must be retrieved from a valid, in-world BlockEntity before instantiation.

```java
// In a hypothetical PlayerInteractionHandler
BlockEntity targetEntity = world.getBlockEntityAt(x, y, z);
ItemContainer container = targetEntity.getComponent(ItemContainer.class);

if (container != null) {
    // Create the window by injecting the block's real inventory
    ContainerBlockWindow window = new ContainerBlockWindow(x, y, z, rot, blockType, container);
    player.getWindowManager().openWindow(window);
}
```

### Anti-Patterns (Do NOT do this)
- **State Desynchronization:** Do not retain a reference to a ContainerBlockWindow instance after it has been closed. Its underlying ItemContainer may be modified by other systems, and the window object will not receive updates.
- **Direct Instantiation:** Do not use `new ContainerBlockWindow()` with a newly created, transient ItemContainer. The window must be linked to a persistent ItemContainer from a BlockEntity to ensure changes are saved.
- **Cross-Thread Modification:** Never call handleAction or access getItemContainer from an asynchronous task or a different thread. All interactions must be marshaled back to the main server thread.

## Data Pipeline
The primary data flow involves receiving a client action, processing it, mutating server state, and notifying the client of the change. The `invalidate()` call is the trigger for this notification.

> Flow:
> Client UI Event (e.g., Sort Button Click) -> `SortItemsAction` Packet -> Server Network Layer -> Player Action Router -> **ContainerBlockWindow.handleAction()** -> ItemContainer.sortItems() -> `this.invalidate()` -> Window Management System -> `WindowUpdate` Packet -> Client UI Re-render

