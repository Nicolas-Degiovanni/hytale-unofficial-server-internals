---
description: Architectural reference for ItemStackTransaction
---

# ItemStackTransaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Value Object (Immutable)

## Definition
```java
// Signature
public class ItemStackTransaction implements Transaction {
```

## Architecture & Concepts
The ItemStackTransaction class is a foundational component of the server-side inventory system. It is not a service or manager, but rather an immutable **Value Object** that represents the complete result of an inventory modification attempt, such as adding or removing an ItemStack from an ItemContainer.

Its primary architectural role is to provide a detailed, atomic record of a completed operation. Instead of returning simple booleans or error codes, inventory methods return an ItemStackTransaction. This provides the calling system with a rich, queryable summary of what changed, what failed, and what items, if any, were left over.

This class embodies the **Composite Pattern**. A single ItemStackTransaction encapsulates a list of more granular ItemStackSlotTransaction objects. This allows for a hierarchical representation of change: the parent object reports the overall success, while the children detail the exact modifications to each individual inventory slot. This design is critical for supporting complex inventory logic, including operations that span multiple slots.

Furthermore, the design explicitly supports **nested or hierarchical inventories** (e.g., a hotbar which is a view into a larger backpack). The `toParent` and `fromParent` methods are transformation functions that remap slot indices between different container coordinate systems, allowing transaction results to be correctly interpreted across different inventory views.

## Lifecycle & Ownership
- **Creation:** An ItemStackTransaction is instantiated exclusively by the inventory system, typically deep within an ItemContainer's implementation, as the final step of an add, remove, or query operation. It is a *result*, not a request. Developers should never create instances of this class directly.

- **Scope:** Extremely transient. An instance typically exists only on the stack for the immediate duration of the inventory method call and its subsequent inspection by the caller. It is not designed to be stored or cached.

- **Destruction:** The object is managed by the Java garbage collector and is eligible for cleanup as soon as all references to it are dropped, which usually happens immediately after the calling logic has processed the result.

## Internal State & Concurrency
- **State:** **Immutable**. All fields are final and are set only once during construction. The internal list of slot transactions is also treated as immutable from the perspective of a consumer. Methods that perform transformations, such as `toParent`, do not modify the instance but instead return a new ItemStackTransaction.

- **Thread Safety:** **Inherently thread-safe**. Due to its immutability, an ItemStackTransaction instance can be safely read by multiple threads without any external synchronization.

    **WARNING:** While the transaction object itself is thread-safe, the inventory systems that *generate* these objects are almost certainly **not**. All modifications to an ItemContainer must be performed from a single, synchronized context, typically the main server thread. Passing a transaction object to another thread for read-only analysis is safe; using it to infer state about a concurrently modified inventory is not.

## API Surface
The public API is designed for inspecting the outcome of an inventory operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| succeeded() | boolean | O(1) | Returns true if the overall operation was considered a success. |
| getQuery() | ItemStack | O(1) | Returns the original ItemStack that was the subject of the inventory operation. |
| getRemainder() | ItemStack | O(1) | Returns the portion of the query that could not be placed in the inventory. Null if the entire stack was consumed. |
| getSlotTransactions() | List | O(1) | Returns the detailed list of individual slot-level changes. |
| wasSlotModified(short slot) | boolean | O(N) | Scans child transactions to determine if a specific slot index was affected. N is the number of slot transactions. |
| toParent(...) | ItemStackTransaction | O(N) | Creates a new transaction by remapping its slot indices to a parent container's coordinate space. |
| fromParent(...) | ItemStackTransaction | O(N) | Creates a new, filtered transaction by remapping slot indices from a parent to a child container's coordinate space. |

## Integration Patterns

### Standard Usage
The correct pattern is to execute an inventory operation and immediately inspect the returned transaction object to determine the outcome and handle any remaining items.

```java
// How a developer should normally use this
ItemContainer inventory = player.getInventory();
ItemStack itemsToAdd = new ItemStack(Item.DIAMOND, 10);

// The transaction is the authoritative result of the operation
ItemStackTransaction transaction = inventory.add(itemsToAdd);

if (transaction.succeeded()) {
    if (transaction.getRemainder() != null) {
        // The inventory is full; drop the remaining items on the ground.
        world.dropItem(player.getPosition(), transaction.getRemainder());
    }
} else {
    // The entire operation failed. The inventory was not modified.
    player.sendMessage("Could not add items to your inventory.");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ItemStackTransaction()`. These objects are strictly internal to the inventory system's implementation and creating them manually will break state assumptions.

- **Ignoring the Result:** A common and critical bug is to call an inventory method and not check the result. This leads to silent failures and item duplication or deletion.
    ```java
    // BAD: This can lead to item loss if the inventory is full.
    inventory.add(itemsToAdd); // The remainder is never checked!
    ```

- **Modifying Internal State:** Do not attempt to modify the list returned by `getSlotTransactions`. It should be treated as a read-only collection.

## Data Pipeline
ItemStackTransaction is not part of a data processing pipeline; it is the terminal output of an inventory operation command.

> Flow:
> Game Logic Command -> ItemContainer.add(ItemStack) -> **Internal Slot Calculation** -> **ItemStackTransaction (Created)** -> Returned to Game Logic for Inspection

