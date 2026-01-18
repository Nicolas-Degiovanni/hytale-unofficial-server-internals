---
description: Architectural reference for MaterialSlotTransaction
---

# MaterialSlotTransaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Transient

## Definition
```java
// Signature
public class MaterialSlotTransaction extends SlotTransaction {
```

## Architecture & Concepts

The MaterialSlotTransaction class is a specialized data transfer object that represents the outcome of an inventory operation initiated by a material-based query, such as "add 50 wood" or "remove 10 iron". It acts as a decorator or wrapper around a lower-level SlotTransaction, enriching it with the context of the original request.

This class is a cornerstone of the server's inventory system, providing a crucial abstraction layer. While SlotTransaction describes the concrete change to a single inventory slot (e.g., "slot 5 changed from empty to 32 wood"), MaterialSlotTransaction provides the high-level narrative: "The request to add 50 wood resulted in this specific slot change, with 18 wood remaining unfulfilled".

This design elegantly separates the *intent* of an operation (the query) from its *execution* (the underlying transaction), allowing game logic to operate in terms of abstract materials without managing the complexities of slot indices, stacking, and partial fulfillment.

### Lifecycle & Ownership
- **Creation:** Instances are not intended for direct instantiation by game logic. They are constructed and returned exclusively by the core inventory management services, typically as the result of a call to a method like `ItemContainer.addMaterial` or `ItemContainer.removeMaterial`.
- **Scope:** Ephemeral. A MaterialSlotTransaction exists only for the duration of the method call that generated it. It is a result object, designed to be immediately inspected by the caller and then discarded.
- **Destruction:** The object is short-lived and becomes eligible for garbage collection as soon as the calling code block completes its evaluation of the transaction result. It holds no persistent references and is not managed by any lifecycle system.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are final and are assigned only once during construction. Methods like `toParent` and `fromParent` do not modify the instance's state; they return a new MaterialSlotTransaction instance with translated data. This immutability guarantees that a transaction result is atomic and cannot be altered after its creation, which is critical for predictable and verifiable inventory logic.

- **Thread Safety:** **Thread-safe**. Due to its immutable nature, a MaterialSlotTransaction instance can be safely read by multiple threads without synchronization.

    **WARNING:** While the object itself is thread-safe, the inventory systems that produce these objects are almost certainly not. All operations that modify inventory state and generate these results must be performed on the appropriate game thread or be protected by explicit locking mechanisms.

## API Surface

The public API provides read-only access to the complete context of the material-based operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | MaterialQuantity | O(1) | Returns the original material and quantity that was requested by the caller. |
| getRemainder() | int | O(1) | Returns the quantity of the material that could not be fulfilled by the transaction. A value of 0 indicates the requested amount was fully satisfied. |
| getTransaction() | SlotTransaction | O(1) | Provides access to the underlying, low-level transaction that was executed on a specific inventory slot. |
| toParent(...) | MaterialSlotTransaction | O(1) | Creates a new transaction instance with slot indices re-calculated relative to a parent container. Essential for nested inventories. |
| fromParent(...) | MaterialSlotTransaction | O(1) | Creates a new transaction instance with slot indices re-calculated from a parent container's coordinate space to a local one. |

## Integration Patterns

### Standard Usage

A developer should never create a MaterialSlotTransaction directly. Instead, they should interact with a container's API and inspect the returned transaction object to understand the outcome.

```java
// Standard pattern for adding items to an inventory and handling partial success.
ItemContainer playerInventory = player.getInventory();
MaterialQuantity woodToGive = new MaterialQuantity(KnownMaterials.WOOD_PLANKS, 100);

// The inventory system performs the operation and returns the result.
MaterialSlotTransaction result = playerInventory.addMaterial(woodToGive);

// Check the outcome.
if (result.succeeded() && result.getRemainder() > 0) {
    // The inventory is full; drop the remaining items on the ground.
    world.dropItem(player.getPosition(), new MaterialQuantity(KnownMaterials.WOOD_PLANKS, result.getRemainder()));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new MaterialSlotTransaction(...)`. This class is an internal result type produced by the inventory system. Constructing it manually bypasses the actual inventory logic and creates a misleading and invalid state representation.

- **Ignoring the Remainder:** Do not solely check `transaction.succeeded()` to determine success. A transaction can succeed in placing *some* items but not all. Always check `getRemainder()` to handle cases where an inventory is full and the operation was only partially completed.

- **Ignoring the Wrapped Transaction:** The `succeeded()` flag on this class is inherited from the wrapped transaction. For complex validation, it may be necessary to inspect the `getSlotBefore()` and `getSlotAfter()` methods on the object returned by `getTransaction()` to fully understand the state change.

## Data Pipeline

MaterialSlotTransaction is the terminal output of a material-based inventory modification pipeline. It encapsulates the final state change for consumption by higher-level game logic.

> Flow:
> Game Logic Request (e.g., `player.giveItem`) -> `ItemContainer.addMaterial(query)` -> Internal Slot Search & Validation -> `SlotTransaction` Execution -> **MaterialSlotTransaction Creation** -> Return to Game Logic -> Logic acts on result (e.g., drop remainder, send network update)

