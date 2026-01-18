---
description: Architectural reference for the Transaction interface, the core contract for inventory operations.
---

# Transaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface Transaction {
   // Methods defined below
}
```

## Architecture & Concepts
The Transaction interface is a foundational contract within the server-side inventory system. It is not a service or a manager, but rather a result object that encapsulates the outcome of a single, atomic inventory operation, such as moving an item between slots or containers.

Its primary architectural role is to decouple the *attempt* of an inventory modification from its *result*. Systems that manipulate inventory do not return simple booleans; they return a Transaction object. This provides the caller with a rich, queryable summary of what changed, whether the operation was successful, and how to translate the operation's context if it spans nested containers.

This pattern is critical for maintaining data integrity, synchronizing client state, and enabling complex inventory features like nested bags or modular equipment. The interface is designed around the principle of immutability; a Transaction represents a historical fact, not a pending change.

### Lifecycle & Ownership
- **Creation:** Implementations of Transaction are instantiated by high-level inventory management systems (e.g., an InventoryManager or ContainerController) at the conclusion of an item manipulation attempt. They are never created directly by game logic handlers.
- **Scope:** A Transaction object is extremely short-lived and transient. Its scope is typically confined to the method in which it was created and its immediate caller. It exists only to communicate the result of one operation.
- **Destruction:** The object is eligible for garbage collection as soon as the calling code has inspected its state and propagated any necessary updates (e.g., sending a network packet). Holding long-term references to Transaction objects is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** As an interface, Transaction defines no state. However, its concrete implementations are expected to be **immutable**. They capture the final state of an operation—such as the set of modified slot indices and the overall success status—at the moment of their creation. They are effectively Data Transfer Objects (DTOs) for inventory results.
- **Thread Safety:** Implementations must be thread-safe for reading. Given their intended immutability and transient nature, they are inherently safe to pass between threads after creation. The systems that *create* Transactions must ensure that the underlying inventory modifications are performed in a thread-safe manner, often by processing all inventory logic on a single game-tick thread.

## API Surface
The public contract of Transaction is focused on querying the result of a completed operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| succeeded() | boolean | O(1) | Returns true if the entire operation completed successfully. This is the primary method for checking the outcome. |
| wasSlotModified(short slotIndex) | boolean | O(1) | Checks if a specific slot index was altered during the operation. Essential for sending minimal state updates to clients. |
| toParent(ItemContainer child, short childSlot, ItemContainer parent) | Transaction | O(1) | **CRITICAL:** Remaps the transaction's context from a child container to its parent. Returns a *new* Transaction object with slot indices adjusted for the parent's perspective. Throws if the relationship is invalid. |
| fromParent(ItemContainer parent, short parentSlot, ItemContainer child) | Transaction | O(1) | **CRITICAL:** Remaps the transaction's context from a parent container to a specific child. This is the inverse of toParent. Returns a *new* Transaction object. Returns null if the parent slot does not correspond to the specified child container. |

## Integration Patterns

### Standard Usage
The canonical use case involves a game logic handler invoking an inventory service, receiving a Transaction, and then acting upon the result.

```java
// A handler processes a player's request to move an item.
ItemContainer source = player.getInventory();
ItemContainer destination = player.getBackpack();

// The manager returns a result object, not a simple boolean.
Transaction result = InventoryManager.moveItem(source, 5, destination, 0);

if (result.succeeded()) {
    // Send a success packet to the client, possibly including
    // only the slots that were modified.
    PacketSender.sendInventoryUpdate(player, result);
} else {
    // Send a failure packet to prevent client desynchronization.
    PacketSender.sendInventoryActionFailed(player);
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to cast a Transaction to a concrete class to modify its state. They represent a completed event and must not be altered.
- **Long-Term Storage:** Storing Transaction objects in a component or cache is incorrect. They are ephemeral and their data may become stale as the inventory changes further.
- **Ignoring Results:** Failing to check `succeeded()` can lead to severe server-client desynchronization, where the server rejects a change but the client believes it was successful.

## Data Pipeline
The Transaction interface serves as a critical data structure that bridges core game logic with the network synchronization layer.

> Flow:
> Client Input Packet -> Network Handler -> Player Command Logic -> InventoryManager.executeMove() -> **Transaction** (Immutable Result) -> Packet Generation Service -> Server Response Packet

