---
description: Architectural reference for ResourceTransaction
---

# ResourceTransaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Transient Data Object

## Definition
```java
// Signature
public class ResourceTransaction extends ListTransaction<ResourceSlotTransaction> {
```

## Architecture & Concepts
ResourceTransaction is an immutable record that encapsulates the complete result of a single, high-level inventory operation, such as adding or removing a quantity of items. It is a foundational component of the server's transactional inventory system, providing a detailed and reliable account of changes within an ItemContainer.

This class does not perform an action; it *is the result* of an action. When game logic requests to add 64 stone to a player's inventory, the inventory system processes this request across multiple slots and then constructs a single ResourceTransaction to summarize the outcome. This object details precisely how much was added (the consumed amount), how much could not be added (the remainder), and which specific slots were affected via a list of child ResourceSlotTransaction objects.

A key architectural feature is its role in managing nested inventories. The methods toParent and fromParent allow the transaction's slot-level details to be remapped between a child container (like a backpack) and its parent (the main player inventory). This enables complex inventory logic while maintaining a consistent and predictable data structure for results.

## Lifecycle & Ownership
- **Creation:** ResourceTransaction instances are created exclusively by the inventory system, typically as the return value from methods like ItemContainer.add or ItemContainer.remove. They are never instantiated directly by high-level game logic.
- **Scope:** Method-scoped. An instance exists only for the duration of the inventory operation's callback or return handling. It is a temporary object used to communicate a result.
- **Destruction:** The object is short-lived and becomes eligible for garbage collection as soon as the calling code finishes processing the result. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** **Immutable**. All fields are set once in the constructor and cannot be changed. Methods that appear to modify the transaction, such as toParent, return a new instance with transformed data. This design guarantees that the record of an inventory change is tamper-proof once created.
- **Thread Safety:** **Fully thread-safe**. Due to its immutability, a ResourceTransaction object can be safely passed across threads without any need for locks or synchronization. It represents a snapshot of a past event, free from race conditions.

## API Surface
The public API is designed for inspecting the outcome of an inventory operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAction() | ActionType | O(1) | Returns the type of action performed (e.g., ADD, REMOVE). |
| getResource() | ResourceQuantity | O(1) | Returns the original resource and quantity requested for the transaction. |
| getConsumed() | int | O(1) | Returns the actual quantity of the resource that was successfully moved. |
| getRemainder() | int | O(1) | Returns the quantity of the resource that could not be moved. |
| toParent(...) | ResourceTransaction | O(N) | Creates a new transaction, remapping slot indices from a child container to its parent's coordinate space. N is the number of slot transactions. |
| fromParent(...) | ResourceTransaction | O(N) | Creates a new transaction, remapping slot indices from a parent container to a specific child's coordinate space. Returns null if no slots are relevant. |

## Integration Patterns

### Standard Usage
The correct pattern is to execute an inventory operation and immediately inspect the returned ResourceTransaction to make decisions.

```java
// Standard pattern for handling an inventory operation result
ItemContainer inventory = player.getInventory();
ResourceQuantity itemsToAdd = new ResourceQuantity("hytale:stone", 32);

ResourceTransaction result = inventory.add(itemsToAdd);

if (result.succeeded()) {
    // Use the result to inform game logic or client updates
    int amountAdded = result.getConsumed();
    int amountLeftOver = result.getRemainder();
    
    player.sendMessage("You picked up " + amountAdded + " stone.");
    if (amountLeftOver > 0) {
        player.sendMessage("Your inventory is full.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ResourceTransaction()`. Doing so creates a synthetic result that does not reflect the actual state of any ItemContainer, leading to severe data desynchronization. This object must only be created by the inventory system itself.
- **Ignoring the Result:** Calling `inventory.add(items)` and not capturing or checking the returned ResourceTransaction is a critical error. This is equivalent to assuming the operation always succeeds, which is never guaranteed and will cause inventory bugs.
- **Retaining References:** Storing a ResourceTransaction instance for later use is incorrect. It represents a state at a single point in time and will become stale as subsequent inventory changes occur. Process it immediately and then discard it.

## Data Pipeline
ResourceTransaction is the terminal output of an internal inventory operation. It serves as the bridge between the inventory system and the rest of the game logic.

> Flow:
> Game Logic Request -> ItemContainer.add/remove -> **ResourceTransaction (Created)** -> Calling System Inspects Result -> Network Message to Client OR Further Game Logic<ctrl63>

