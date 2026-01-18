---
description: Architectural reference for MoveTransaction
---

# MoveTransaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Value Object / Transient

## Definition
```java
// Signature
public class MoveTransaction<T extends Transaction> implements Transaction {
```

## Architecture & Concepts

The MoveTransaction class is a foundational component of the server-side inventory system. It is an immutable value object that represents the outcome of a compound inventory operation: moving an item stack from one container to another. It does not perform the move itself; rather, it serves as a detailed, atomic record of what occurred after an attempt to move an item.

This class acts as a composite transaction, encapsulating two distinct sub-transactions:
1.  A **removeTransaction**, which is always a SlotTransaction detailing the removal of items from the source slot.
2.  An **addTransaction**, a generic transaction of type T, detailing the addition of items to the destination container. The generic nature allows it to represent simple single-slot additions or more complex multi-slot additions.

A critical concept is the **MoveType** enum, which defines the direction of the transaction from the perspective of the container that generated it.
-   **MOVE_TO_SELF**: An item was moved *from* an external container *into* the container that owns this transaction.
-   **MOVE_FROM_SELF**: An item was moved *from* the container that owns this transaction *to* an external container.

This directional context is essential for higher-level systems to correctly interpret the transaction and synchronize state across different inventory views.

## Lifecycle & Ownership

-   **Creation:** A MoveTransaction is instantiated exclusively by the core inventory logic, typically within an ItemContainer or a related service, as the return value of an item movement operation. It is never created directly by application-level code. It is a *result*, not a *command*.
-   **Scope:** Its lifetime is extremely short. It exists ephemerally on the call stack for the duration of a single inventory update tick. It is used to communicate the result of the operation to the caller and is then immediately eligible for garbage collection.
-   **Destruction:** The object is managed by the Java Garbage Collector and is reclaimed as soon as it falls out of scope, which is typically moments after its creation.

## Internal State & Concurrency

-   **State:** The MoveTransaction is **deeply immutable**. All of its fields are final and are set only during construction. Methods like toParent and fromParent do not modify the instance; they return a new MoveTransaction instance with transformed data. This design guarantees that a transaction record cannot be altered after its creation, ensuring data integrity.
-   **Thread Safety:** This class is **inherently thread-safe** due to its immutability. An instance can be safely shared and read across multiple threads without any risk of data corruption or need for external synchronization.

    **WARNING:** While the MoveTransaction object itself is thread-safe, the inventory system that creates and processes it is not. All modifications to core inventory state must be performed on the main server thread to prevent race conditions.

## API Surface

The public API provides a read-only view into the results of the move operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| succeeded() | boolean | O(1) | Returns true if the entire move operation was successful. |
| getRemoveTransaction() | SlotTransaction | O(1) | Retrieves the transaction for the item removal part of the move. |
| getAddTransaction() | T | O(1) | Retrieves the transaction for the item addition part of the move. |
| getMoveType() | MoveType | O(1) | Returns the direction of the move relative to the originating container. |
| getOtherContainer() | ItemContainer | O(1) | Gets a reference to the other container involved in the transaction. |
| wasSlotModified(short slot) | boolean | O(1) | Checks if a given slot index was affected by either the add or remove sub-transaction. |
| toParent(...) | MoveTransaction | O(1) | Creates a new transaction with slot indices remapped to a parent container's coordinate space. |
| fromParent(...) | MoveTransaction | O(1) | Creates a new transaction by remapping slot indices from a parent container's space to a local one. |

## Integration Patterns

### Standard Usage

A MoveTransaction is not created directly. It is received as a result from an ItemContainer method call. The developer then inspects its properties to understand the outcome and update other game systems accordingly.

```java
// How a developer should normally use this
// Assume 'playerInventory' and 'chest' are ItemContainer instances
MoveTransaction result = playerInventory.moveFrom(slotIndex, amount, chest);

if (result.succeeded()) {
    // The move was successful.
    // The 'result' object contains the details of which slots changed.
    // This information can be used to send precise network updates to the client.
    SlotTransaction removal = result.getRemoveTransaction();
    Transaction addition = result.getAddTransaction();
    // ... process sub-transactions
} else {
    // The move failed. No state was changed.
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new MoveTransaction(...)`. These objects are carefully constructed by the inventory system to reflect actual state changes. Manual creation will lead to a state mismatch between the transaction record and the real inventory.
-   **Ignoring MoveType:** Failing to check the MoveType will lead to incorrect state interpretation. You must use it to determine whether the `removeTransaction` happened in the local container or the `otherContainer`.
-   **Modifying via Reflection:** Attempting to modify the final fields of a MoveTransaction is a severe violation of the system's contract and will cause unpredictable and difficult-to-debug inventory corruption.

## Data Pipeline

The MoveTransaction serves as a structured data record within the server's inventory processing pipeline. It translates a high-level action into a verifiable result.

> Flow:
> Player Action (e.g., Drag/Drop UI Event) -> Network Command -> Server Inventory Service -> `ItemContainer.moveFrom()` -> **MoveTransaction Created** -> Inventory Service analyzes result -> Network Packet with state changes -> Client Inventory System -> UI Update

