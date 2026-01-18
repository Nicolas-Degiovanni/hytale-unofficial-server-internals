---
description: Architectural reference for CombinedItemContainer
---

# CombinedItemContainer

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Transient

## Definition
```java
// Signature
public class CombinedItemContainer extends ItemContainer {
```

## Architecture & Concepts
The CombinedItemContainer is an implementation of the **Composite** design pattern. It acts as a virtual, unified facade over an array of one or more child ItemContainer instances. This allows disparate inventories, such as a player's main inventory, hotbar, and armor slots, to be treated as a single, contiguous inventory space by higher-level game systems.

The primary architectural function of this class is to translate operations from a virtual, combined slot index to a physical slot index within one of its child containers. For example, a call to get an item from slot 40 might be routed to slot 4 of the second child container, after accounting for the capacity of the first.

A critical and complex feature is its handling of atomic operations across multiple containers. The `readAction` and `writeAction` methods implement a recursive, nested locking strategy. When an action is performed, this class acquires a lock on the first child container, then the second, and so on, until all required locks are held. Only then is the action executed. This guarantees that operations spanning container boundaries are atomic, preventing race conditions and data corruption.

Event propagation is also managed. The CombinedItemContainer listens for change events from all its children, translates the transaction data (such as slot indices) back into its own virtual coordinate space, and then re-emits the event as if it originated from itself.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, `new CombinedItemContainer(ItemContainer... containers)`. It is typically created by a higher-level manager, such as a PlayerEntity or a complex block entity, to aggregate its various inventory components.
- **Scope:** The lifecycle of a CombinedItemContainer is strictly bound to its owner. It persists as long as the owning entity (e.g., a player) exists in the world. It does not persist across sessions unless its state is explicitly serialized by an owning system.
- **Destruction:** The object is eligible for garbage collection once the owning entity is destroyed and all external references to it are released. It has no explicit `destroy` or `close` method.

## Internal State & Concurrency
- **State:** The internal state consists of a `final` array of child ItemContainers. While the composition of the container is immutable after creation, the contents of the underlying child containers are mutable. This class does not cache any item data; all read/write operations are delegated directly to the children in real-time.
- **Thread Safety:** This class is thread-safe. All read and write operations must be wrapped in `readAction` or `writeAction` lambdas, which are inherited from the ItemContainer parent. These methods ensure safety by creating a nested chain of lock acquisitions across all child containers in a deterministic order.

    **WARNING:** The nested locking mechanism, while ensuring atomicity, can become a performance bottleneck if the number of combined containers is large. It also introduces a theoretical risk of deadlock if other systems attempt to lock the same containers in a different order.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getContainerForSlot(short slot) | ItemContainer | O(N) | Iterates through child containers to find which one owns the specified virtual slot. N is the number of child containers. |
| getCapacity() | short | O(N) | Calculates the total capacity by summing the capacities of all child containers. Avoid calling this in performance-critical loops. |
| registerChangeEvent(...) | EventRegistration | O(N) | Registers a listener on this container and all of its children, returning a combined registration object for proper unsubscription. |
| clone() | CombinedItemContainer | N/A | Throws UnsupportedOperationException. This object is not designed to be cloned. |
| setGlobalFilter(FilterType) | void | N/A | Throws UnsupportedOperationException. Filters must be applied to the underlying child containers directly. |
| containsContainer(ItemContainer) | boolean | O(N) | Recursively checks if the specified container is this container or one of its children. |

## Integration Patterns

### Standard Usage
A CombinedItemContainer should be used to present a simplified, single interface to a complex set of inventories. The client code does not need to be aware of the underlying container boundaries.

```java
// Assume player.getInventory() and player.getHotbar() return ItemContainers
ItemContainer mainInventory = player.getInventory();
ItemContainer hotbar = player.getHotbar();

// Create a unified view of the inventory and hotbar
CombinedItemContainer fullInventory = new CombinedItemContainer(mainInventory, hotbar);

// Interact with the full inventory using virtual slot indices
ItemStack firstHotbarItem = fullInventory.getSlot(mainInventory.getCapacity());
```

### Anti-Patterns (Do NOT do this)
- **Unsupported Operations:** Do not call `clone()` or `setGlobalFilter()`. These operations are explicitly not supported and will throw an exception. Global filters must be set on the child containers individually.
- **Performance Blindness:** Avoid calling iterative methods like `getCapacity()` or `getContainerForSlot()` inside tight game loops. Cache the capacity if it is needed frequently.
- **State Mismanagement:** Do not modify the child containers directly from one thread while another thread is performing a `writeAction` on the CombinedItemContainer. Always use the combined container's `writeAction` to ensure proper locking.

## Data Pipeline
The primary data flow involves the translation of virtual slot indices and the propagation of events. A write operation follows this logical path:

> Flow:
> External System Call (e.g., `setSlot(40, item)`) -> **CombinedItemContainer** -> Virtual-to-Physical Slot Translation -> Child Container `setSlot(4, item)` -> Child Container emits `ItemContainerChangeEvent` -> **CombinedItemContainer** listener intercepts event -> Event data is translated back to virtual coordinates -> **CombinedItemContainer** emits its own `ItemContainerChangeEvent` -> External System Listeners are notified.

