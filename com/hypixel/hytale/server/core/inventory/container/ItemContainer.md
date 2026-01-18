---
description: Architectural reference for ItemContainer
---

# ItemContainer

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Component (Abstract Base)

## Definition
```java
// Signature
public abstract class ItemContainer {
```

## Architecture & Concepts

The ItemContainer is the foundational server-side abstraction for any object capable of holding items. It serves as the definitive model for player inventories, chests, crafting tables, and other in-game containers. It is not a concrete implementation but rather a contract that defines the behavior and interactions for all inventory systems.

The architecture is built upon three core principles:

1.  **Transactional Operations:** Every state modification is designed as a transaction. Methods like `addItemStack` or `moveItemStackFromSlotToSlot` do not simply alter state and return void; they return a detailed **Transaction** object. This object encapsulates the success or failure of the operation, the items before and after, and any remainder. This atomic approach is critical for ensuring data consistency, especially in complex multi-step operations like crafting or trading.

2.  **Event-Driven Updates:** ItemContainer employs an Observer pattern through its internal `SyncEventBusRegistry`. Upon the successful completion of any transaction, the `sendUpdate` method is invoked, which dispatches an `ItemContainerChangeEvent`. This decouples the inventory system from its consumers. Other systems, such as the network layer responsible for client synchronization or a quest system tracking objectives, can subscribe to these events without being tightly coupled to the container's implementation.

3.  **Deferred Concurrency Control:** The class is abstract and delegates locking and thread safety to its concrete implementations through the `readAction` and `writeAction` methods. This is a form of the Template Method pattern. The public API wraps all state-mutating logic within `writeAction` and all read-only logic within `readAction`. This guarantees that any concrete subclass, such as a `ConcurrentItemContainer`, can implement robust locking (e.g., using `ReentrantReadWriteLock`) without altering the public contract.

Complex inventory logic, such as stacking rules or material-based removal, is delegated to a suite of internal utility classes (e.g., `InternalContainerUtilItemStack`). This maintains a clean separation of concerns, allowing ItemContainer to focus on state management, concurrency, and event dispatching.

### Lifecycle & Ownership

-   **Creation:** An ItemContainer is never instantiated directly. Concrete subclasses are created by their logical owners. For example, a `PlayerEntity` creates its corresponding `PlayerInventory` upon initialization. A `ChestBlockEntity` creates its `ChestContainer` when it is loaded into the world.
-   **Scope:** The lifetime of an ItemContainer is strictly tied to its owner. It persists as long as the owning entity or block entity exists in the game session.
-   **Destruction:** There is no explicit destruction or `dispose` method. When the owner is destroyed (e.g., a player logs out, a chest is broken), the ItemContainer instance loses all strong references and becomes eligible for standard Java garbage collection.

## Internal State & Concurrency

-   **State:** The state of an ItemContainer is inherently **mutable**. Its primary purpose is to manage a dynamic collection of ItemStack objects within a fixed capacity. It does not perform any caching beyond holding the item collection itself.

-   **Thread Safety:** This abstract class is **not** thread-safe on its own. However, it is designed to enforce thread safety in all of its concrete implementations. The `readAction` and `writeAction` contract requires subclasses to provide a locking mechanism. All public methods correctly use these wrappers, ensuring that calls to a concrete implementation are thread-safe.

    **WARNING:** Developers creating new subclasses of ItemContainer *must* provide a correct and robust implementation for `readAction` and `writeAction` to prevent race conditions and data corruption.

## API Surface

The public API is designed around high-level, transactional inventory operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addItemStack(itemStack) | ItemStackTransaction | O(N) | Adds an ItemStack to the container, first attempting to stack with existing items, then filling empty slots. Returns a transaction detailing the result and any remaining items. |
| setItemStackForSlot(slot, itemStack) | ItemStackSlotTransaction | O(1) | Directly sets or replaces the ItemStack in a specific slot. Bypasses stacking logic. |
| removeItemStackFromSlot(slot, quantity) | ItemStackSlotTransaction | O(1) | Removes a specific quantity of an item from a single slot. |
| moveItemStackFromSlotToSlot(slot, quantity, containerTo, slotTo) | MoveTransaction | O(1) | Atomically moves a quantity of items from a slot in this container to a slot in another. Handles stacking, swapping, and filtering. This is the primary primitive for user-driven inventory management. |
| clear() | ClearTransaction | O(N) | Removes all ItemStacks from the container. |
| registerChangeEvent(consumer) | EventRegistration | O(1) | Registers a listener that is invoked after any successful state change. |
| toPacket() | InventorySection | O(N) | Serializes the container's state into a network packet for client synchronization. |

*Complexity is denoted where N is the capacity of the container.*

## Integration Patterns

### Standard Usage

Interaction with an ItemContainer should always be through a reference obtained from its owner (e.g., a Player or Block Entity). All modifications should be treated as transactional operations, and the returned transaction object should be inspected if the outcome is important.

```java
// How a developer should normally use this
Player player = getPlayer();
ItemContainer inventory = player.getInventory();

ItemStack itemsToAdd = new ItemStack("hytale:stone", 10);
ItemStackTransaction transaction = inventory.addItemStack(itemsToAdd);

if (!transaction.succeeded()) {
    // Handle the case where the inventory is full
    ItemStack remainder = transaction.getRemainder();
    world.dropItem(player.getPosition(), remainder);
}
```

### Anti-Patterns (Do NOT do this)

-   **Calling Internal Methods:** Never call any `internal_` prefixed method directly. These methods bypass the crucial `writeAction` lock and the `sendUpdate` event dispatch. Doing so will lead to race conditions and desynchronization between the server and client.
-   **Ignoring Transaction Results:** Do not assume an operation will succeed. Always check the `succeeded()` method on the returned transaction if the operation's success is critical for subsequent game logic. Failure to do so can lead to item duplication or deletion bugs.
-   **Incorrect Subclassing:** When extending ItemContainer, failing to implement a proper locking mechanism in `readAction` and `writeAction` will break the thread-safety contract for the entire inventory system.

## Data Pipeline

The flow of data for a typical write operation demonstrates the transactional and event-driven nature of the system.

> Flow:
> Game Logic Call (e.g., Player Interaction) -> **ItemContainer.moveItemStackFromSlotToSlot()** -> `writeAction` (Acquires Lock) -> `internal_move...` (State Change) -> Transaction Object Created -> `sendUpdate(transaction)` -> `ItemContainerChangeEvent` Dispatched -> Event Bus -> Network System (sends packet to client) & Other Game Systems (e.g., Quests)

