---
description: Architectural reference for ItemContainerWindow
---

# ItemContainerWindow

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Contract Interface

## Definition
```java
// Signature
public interface ItemContainerWindow {
```

## Architecture & Concepts
The ItemContainerWindow interface defines a fundamental contract for any server-side object that represents a user-interface "window" capable of holding items. It serves as a crucial abstraction layer, decoupling systems that need to manipulate inventory data from the specific type of UI or game object they are interacting with.

This interface embodies the **Adapter Pattern**. It provides a standardized method, getItemContainer, to access the underlying data model (the ItemContainer) of various disparate window types, such as a player's main inventory, a storage chest, or a crafting grid. By programming against this interface, server systems like item transfer logic, crafting validators, or loot droppers can operate generically without needing to know the concrete details of the window they are affecting.

This design promotes modularity and extensibility. A new type of block with storage, for example, can be added to the game by simply creating a new class that implements ItemContainerWindow, without requiring any changes to the core systems that manage item interactions.

## Lifecycle & Ownership
As an interface, ItemContainerWindow has no lifecycle of its own. The lifecycle is entirely dictated by the concrete classes that implement it.

- **Creation:** Implementing objects (e.g., a ChestWindow or PlayerInventoryWindow) are typically instantiated when a player interacts with a corresponding game element. For instance, a ChestWindow is created when a player opens a chest block in the world.
- **Scope:** The lifetime of an implementing object is tied to the duration of the interaction. A ChestWindow exists only as long as the player has the chest UI open. A PlayerInventoryWindow, however, may persist for the entire player session.
- **Destruction:** Instances are eligible for garbage collection once the interaction that created them concludes and all references are released. For example, when a player closes the chest UI, the server-side ChestWindow object is typically destroyed.

**WARNING:** Systems holding a reference to an ItemContainerWindow must be aware of its potentially transient nature and should not cache it across distinct user interactions.

## Internal State & Concurrency
The interface itself is stateless. However, it provides access to a stateful object, the ItemContainer.

- **State:** All state is managed by the implementing class, primarily within the ItemContainer it returns. This container holds the mutable collection of items.
- **Thread Safety:** This interface makes no guarantees about thread safety. It is the responsibility of the **implementing class** to ensure that modifications to the underlying ItemContainer are atomic and safe from race conditions. Server logic often involves multiple threads (e.g., network thread, main game tick thread), and concurrent access to an inventory is a common scenario. Implementations are expected to use appropriate synchronization mechanisms, such as locks on the ItemContainer instance, to prevent data corruption.

## API Surface
The public contract is minimal, designed for a single, focused purpose.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemContainer() | @Nonnull ItemContainer | O(1) | Retrieves the backing data container for this window. This method must never return null. |

## Integration Patterns

### Standard Usage
The primary pattern is to obtain a reference to an ItemContainerWindow from a higher-level object (like a Player entity) and use it to access the inventory for manipulation. This allows for generic, reusable inventory logic.

```java
// How a server system should interact with any window
void transferFirstItem(ItemContainerWindow source, ItemContainerWindow destination) {
    ItemContainer sourceContainer = source.getItemContainer();
    ItemContainer destContainer = destination.getItemContainer();

    // Generic logic to move an item, unaware of window types
    ItemStack stack = sourceContainer.getStackInSlot(0);
    if (!stack.isEmpty()) {
        destContainer.addItem(stack);
        sourceContainer.setStackInSlot(0, ItemStack.EMPTY);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Type Casting:** Avoid casting an ItemContainerWindow to a concrete implementation. This violates the principle of the abstraction and creates brittle code that will break when new window types are introduced.
  ```java
  // BAD: Defeats the purpose of the interface
  if (window instanceof ChestWindow) {
      ChestWindow chest = (ChestWindow) window;
      // ... custom logic
  }
  ```
- **Assuming Implementation Details:** Do not assume the returned ItemContainer is a specific subclass or has a certain size. Always query the container for its properties, such as `getSlots()`.

## Data Pipeline
This interface is a key component in the server's item management pipeline, typically acting as the bridge between network commands and the core inventory data model.

> Flow:
> Client UI Action (e.g., drag-and-drop item) -> Network Packet -> Server Packet Handler -> **ItemContainerWindow** -> ItemContainer -> State Change -> Network Update Packet -> Client UI Update

