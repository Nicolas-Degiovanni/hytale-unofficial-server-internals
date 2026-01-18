---
description: Architectural reference for ListTransaction
---

# ListTransaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Value Object / Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ListTransaction<T extends Transaction> implements Transaction {
```

## Architecture & Concepts
The ListTransaction class is a foundational component of the server's inventory system, implementing the **Composite Design Pattern** for inventory operations. It acts as a container that aggregates multiple individual Transaction objects into a single, logical unit.

By implementing the Transaction interface itself, a ListTransaction can be treated identically to a single, atomic transaction. This allows for the construction of complex, hierarchical inventory operations (e.g., a transaction containing other list-based transactions) without complicating the consuming code.

Its primary purpose is to represent the collective outcome of a high-level inventory action that results in several discrete changes. For example, a "craft item" action might produce a ListTransaction containing one transaction for consuming ingredients and another for adding the crafted item to the player's inventory.

The core architectural feature is the separation of the overall success state from the individual sub-transaction states. A ListTransaction can be marked as failed even if some of its constituent transactions succeeded, providing a single, authoritative result for the entire operation.

## Lifecycle & Ownership
- **Creation:** ListTransaction instances are ephemeral and created on-demand by higher-level inventory management services. They are typically returned as the result of a method call that modifies an ItemContainer, such as moving, swapping, or crafting items. The static factory methods `getEmptyTransaction` are used for no-op results.
- **Scope:** Extremely short-lived. An instance exists only within the scope of the inventory operation that created it. It is a result object, not a persistent entity, and is intended to be inspected immediately and then discarded.
- **Destruction:** The object holds no external resources and is managed entirely by the Java garbage collector. It becomes eligible for collection as soon as it is no longer referenced, typically upon exiting the method that received it.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, including the `succeeded` flag and the list of sub-transactions, is set at construction time and cannot be modified. The constructor defensively wraps the provided list using `Collections.unmodifiableList` to guarantee this contract. This design ensures that a transaction result is stable and cannot be accidentally altered after its creation.
- **Thread Safety:** **Fully thread-safe**. Due to its immutability, a ListTransaction instance can be safely passed between threads without any external synchronization. Multiple threads can read its state concurrently without risk of data corruption or race conditions.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEmptyTransaction(boolean) | ListTransaction | O(1) | Static factory. Returns a shared, empty transaction instance representing success or failure. |
| succeeded() | boolean | O(1) | Returns the canonical success state of the entire aggregated operation. |
| wasSlotModified(short) | boolean | O(N) | Checks if any successful sub-transaction modified the specified slot. N is the number of sub-transactions. |
| getList() | List<T> | O(1) | Returns an unmodifiable view of the underlying sub-transactions. |
| toParent(...) | ListTransaction | O(N) | Creates a new ListTransaction by re-mapping the slot coordinates of all sub-transactions to a parent container's coordinate space. |
| fromParent(...) | ListTransaction | O(N) | Creates a new ListTransaction by filtering and re-mapping sub-transactions from a parent container's coordinate space. Returns null if no transactions apply. |

## Integration Patterns

### Standard Usage
A ListTransaction is typically consumed as a return value from an inventory service. The consumer should first check the overall success state before processing the individual changes.

```java
// An inventory service performs a complex operation
ListTransaction<Transaction> result = inventory.moveAllItems(source, destination);

// Always check the overall success flag first
if (result.succeeded()) {
    // If successful, iterate through sub-transactions to update clients or other systems
    for (Transaction subTx : result.getList()) {
        if (subTx.wasSlotModified(TARGET_SLOT)) {
            // ... logic to handle specific slot update
        }
    }
} else {
    // The entire operation failed; do not process sub-transactions
    log.warn("Failed to move items between containers.");
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Overall Success:** Do not iterate the sub-transaction list without first checking `succeeded()`. If the overall transaction failed, the state of sub-transactions is undefined and should not be trusted for game logic.
- **Attempting to Modify the List:** The list returned by `getList()` is immutable. Any attempt to add, remove, or clear its elements will throw an `UnsupportedOperationException`. Transactions are results, not builders.
- **Relying on Sub-Transaction Success:** Do not assume that if `succeeded()` is true, all sub-transactions also succeeded. The composite success is a distinct logical state.

## Data Pipeline
A ListTransaction serves as a structured data carrier that aggregates the results of an inventory operation before they are propagated to other systems, such as the network layer.

> Flow:
> Player Action -> InventoryManager.executeMove() -> Multiple ItemContainer changes -> **ListTransaction** (created) -> Network Packet Encoder -> Client-side Inventory Update

