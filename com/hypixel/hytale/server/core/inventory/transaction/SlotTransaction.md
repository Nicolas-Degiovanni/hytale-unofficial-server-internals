---
description: Architectural reference for SlotTransaction
---

# SlotTransaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Value Object / Data Transfer Object

## Definition
```java
// Signature
public class SlotTransaction implements Transaction {
```

## Architecture & Concepts

The SlotTransaction is an immutable data record that represents the outcome of a single, atomic operation on a specific slot within an ItemContainer. It is a fundamental building block of the server's inventory system, acting as a "receipt" that provides a complete, verifiable history of a state change.

This class is not a service or a manager; it holds no logic for performing inventory modifications. Instead, it is created and returned by methods that *do* perform modifications, such as `ItemContainer.addItem` or `ItemContainer.swapSlots`. Its primary purpose is to decouple the act of changing inventory state from the logic that needs to react to that change, such as network synchronization or gameplay event triggers.

A key architectural feature revealed by the `toParent` and `fromParent` methods is the support for hierarchical or composite inventories. The system allows an ItemContainer to be nested within another. These methods provide the necessary coordinate space translation, allowing a transaction that occurred in a child container (e.g., a backpack) to be understood in the context of the parent container (e.g., the player's main inventory).

## Lifecycle & Ownership

-   **Creation:** SlotTransaction instances are created exclusively by low-level inventory management services, typically an `ItemContainer`, as the direct result of an attempted modification. They are never instantiated by high-level game logic. The static `FAILED_ADD` instance is used as a flyweight to prevent unnecessary object allocation for a common failure case.

-   **Scope:** These objects are highly transient and short-lived. Their scope is typically confined to the method call that generated them and any immediate downstream consumers. They are not intended to be cached or stored long-term.

-   **Destruction:** The object is eligible for garbage collection as soon as the inventory operation and any subsequent reactions (e.g., sending a network packet) are complete. There is no manual destruction or cleanup process.

## Internal State & Concurrency

-   **State:** **Immutable**. All fields are declared `final` and are set only once within the constructor. Once a SlotTransaction is created, its state can never be changed. This design guarantees that the record of the inventory change is stable and cannot be accidentally modified by consumers.

-   **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a SlotTransaction instance can be safely passed between threads without any need for locks, synchronization, or defensive copies. This is a critical property for a high-performance, multi-threaded server environment where inventory state may be read by multiple systems concurrently.

## API Surface

The public API is designed for inspecting the result of an operation. It provides read-only access to the pre- and post-operation state of the affected slot.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| succeeded() | boolean | O(1) | Returns true if the underlying inventory operation was successful. This is the primary field to check before processing other data. |
| wasSlotModified(short slot) | boolean | O(1) | Checks if this transaction corresponds to the specified slot index. |
| toParent(parent, start, container) | SlotTransaction | O(1) | Creates a new transaction with its slot index translated into the coordinate space of a parent container. Essential for nested inventories. |
| fromParent(parent, start, container) | SlotTransaction | O(1) | Creates a new transaction by translating a parent-space slot index into the local coordinate space of a child container. Returns null if the slot is not within the child's bounds. |

## Integration Patterns

### Standard Usage

The intended use is to receive a SlotTransaction as a return value from an inventory modification method. The caller then inspects the transaction to determine the outcome and propagate changes to other systems, such as the network layer.

```java
// A service attempts to add an item to a player's inventory
ItemContainer playerInventory = player.getInventory();
ItemStack itemToAdd = new ItemStack(Material.IRON_INGOT, 10);

// The transaction object captures the result
SlotTransaction result = playerInventory.addItem(itemToAdd);

// The caller inspects the result to react accordingly
if (result.succeeded()) {
    // Use the transaction data to build a precise network update
    PacketSender.queue(new ClientboundInventoryUpdatePacket(
        result.getSlot(),
        result.getSlotAfter()
    ));
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create a SlotTransaction using `new SlotTransaction()`. Doing so creates a "fake" result that does not reflect the actual state of any ItemContainer. This will lead to severe data corruption and desynchronization between the server and clients. Transactions must only be generated by the inventory system itself.

-   **Ignoring the Result:** Calling an inventory modification method and discarding the returned SlotTransaction is a critical bug. The caller has no way of knowing if the operation succeeded, how the inventory was changed, or what state needs to be synchronized with the client. Always check the `succeeded` flag.

-   **State Assumption:** Do not assume an operation succeeded and then read from the ItemContainer directly. The returned SlotTransaction is the sole source of truth for what occurred during that specific operation.

## Data Pipeline

A SlotTransaction serves as a data carrier that flows from the core inventory system outwards to other parts of the server application.

> Flow:
> Gameplay Logic -> `ItemContainer.modifySlot()` -> **SlotTransaction** (Generated Result) -> Event Bus / Network Service -> `ClientboundInventoryUpdatePacket` -> Client Rendering

---

