---
description: Architectural reference for TagTransaction
---

# TagTransaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Transient

## Definition
```java
// Signature
public class TagTransaction extends ListTransaction<TagSlotTransaction> {
```

## Architecture & Concepts
The TagTransaction class is a fundamental data structure within the server-side inventory system. It functions as an immutable record, or "receipt," that encapsulates the complete outcome of a high-level inventory operation, such as adding or removing items from a logical group of slots.

Architecturally, it sits at the intersection of inventory logic and game systems. It is not an active component that performs work; rather, it is the data-rich result produced by services like an InventoryManager. Its primary design purpose is to provide a detailed, hierarchical summary of a change that may have affected multiple inventory slots simultaneously.

A TagTransaction is a composite object. It represents a single, coarse-grained operation (e.g., "add items to the hotbar") while containing a list of fine-grained TagSlotTransaction objects. Each nested TagSlotTransaction details the precise before-and-after state of an individual slot, providing a complete audit trail. This two-tier structure is critical for supporting complex game logic, such as partial item stacking or atomic "all or nothing" transfers.

The methods `toParent` and `fromParent` are key to supporting nested inventories (e.g., a backpack inside a player's main inventory). They provide the coordinate transformation logic necessary to translate transaction details between a child container and its parent, ensuring that inventory updates are correctly resolved and synchronized across complex container hierarchies.

## Lifecycle & Ownership
- **Creation:** A TagTransaction is instantiated exclusively by internal inventory management services at the conclusion of an item manipulation attempt. It is never created directly by high-level game logic code. It is the *result* of an operation, not the initiator.

- **Scope:** Extremely short-lived and method-scoped. An instance typically exists only on the call stack, passed as a return value from an inventory service to the original caller (e.g., a network packet handler).

- **Destruction:** The object holds no external resources and is not registered with any persistent service. It becomes eligible for garbage collection as soon as the calling method finishes inspecting its state, such as checking the success flag or the remainder count.

## Internal State & Concurrency
- **State:** **Immutable**. All fields are final and are definitively set during construction. The internal list of slot transactions is populated once and is not intended for external modification. This immutability guarantees that a transaction record cannot be altered after its creation, making it a reliable source of truth for a past event.

- **Thread Safety:** **Inherently thread-safe for reads**. Due to its immutable design, a TagTransaction instance can be safely passed across threads and read without any synchronization mechanisms.

    **WARNING:** While the object itself is thread-safe, the inventory operations that *generate* it are not. All modifications to an ItemContainer that result in a TagTransaction must be performed within a synchronized block or from a single, dedicated thread to prevent severe data corruption and item duplication bugs.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAction() | ActionType | O(1) | Returns the type of operation performed, such as ADD or REMOVE. |
| getTagIndex() | int | O(1) | Returns the numerical identifier for the logical slot group (tag) that was targeted. |
| getRemainder() | int | O(1) | Reports how many items could not be processed in the transaction. A non-zero value indicates a partial success or total failure. |
| isAllOrNothing() | boolean | O(1) | Indicates if the operation was configured to be atomic. If true, the transaction only succeeds if all items can be moved. |
| toParent(...) | TagTransaction | O(N) | Creates a new transaction by remapping its slot indices to a parent container's coordinate space. Critical for nested inventories. |
| fromParent(...) | TagTransaction | O(N) | Creates a new transaction by filtering and remapping slot data from a parent context to a child. Returns null if no contained slots are relevant. |

## Integration Patterns

### Standard Usage
A TagTransaction should be treated as a read-only result object. The typical pattern is to invoke a high-level inventory service, receive a TagTransaction, and then inspect its properties to determine the next steps in game logic.

```java
// A hypothetical service processes a player's request to pick up items.
ItemContainer playerInventory = player.getInventory();
ItemStack itemsToPickup = new ItemStack(Material.STONE, 20);

// The manager returns a detailed record of the operation.
TagTransaction result = InventoryManager.addItemsByTag(playerInventory, "MAIN_STORAGE", itemsToPickup);

if (!result.succeeded()) {
    int itemsNotAdded = result.getRemainder();
    if (itemsNotAdded > 0) {
        // Drop the remaining items back into the world or notify the player.
        World.dropItemStack(player.getPosition(), new ItemStack(Material.STONE, itemsNotAdded));
    }
}
// The 'result' object is now out of scope and will be garbage collected.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new TagTransaction()`. Doing so creates a synthetic and disconnected record that does not reflect any actual change in an inventory. This will lead to desynchronization between game state and game logic.

- **Ignoring the Remainder:** A common and critical bug is to call an "add" operation and fail to check `getRemainder()`. If the inventory is full, the remainder will be non-zero. Ignoring it will cause the calling code to assume all items were added, effectively deleting the remainder from the game world.

- **State Mutation:** Do not attempt to retrieve the internal list of slot transactions and modify it. The object is a historical record; altering it is akin to rewriting the past and will break the logical consistency of the inventory system.

## Data Pipeline
The TagTransaction is a data product, not a processor. Its place in the pipeline is as an output that informs subsequent actions.

> Flow:
> Network Packet (Player Input) -> InventoryRequestHandler -> InventoryManager -> **TagTransaction (Created)** -> InventoryRequestHandler (Result Inspected) -> Network Packet (Response to Client) or World Update (e.g., Drop Remaining Items)

