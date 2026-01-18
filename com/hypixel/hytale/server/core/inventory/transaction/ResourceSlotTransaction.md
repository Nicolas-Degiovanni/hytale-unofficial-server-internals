---
description: Architectural reference for ResourceSlotTransaction
---

# ResourceSlotTransaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Value Object

## Definition
```java
// Signature
public class ResourceSlotTransaction extends SlotTransaction {
```

## Architecture & Concepts
The ResourceSlotTransaction is an immutable data record that represents the outcome of an attempt to add or remove a specific *quantity* of a resource from a single inventory slot. It is a specialized subclass of SlotTransaction, designed for operations where the exact ItemStack is less important than the amount of a resource it represents. For example, a request to "take 10 oak planks" would generate a ResourceSlotTransaction, whereas a request to "move this specific enchanted sword" would use a more general SlotTransaction.

This class is a fundamental component of the server's inventory system, acting as a precise receipt for a completed operation. Its primary role is to communicate the results of a transaction—how much was requested, how much was actually consumed, and what remains—back to the calling system.

The presence of the *toParent* and *fromParent* methods indicates a sophisticated inventory architecture that supports nested or composite containers. These methods allow transaction results to be re-contextualized as they propagate up or down a hierarchy of containers, such as a player's backpack being a container within their main inventory.

## Lifecycle & Ownership
- **Creation:** A ResourceSlotTransaction is instantiated exclusively by the inventory management system, typically within an ItemContainer, as the direct result of a resource manipulation call (e.g., *takeResource* or *addResource*). It is never created directly by game logic.
- **Scope:** This object is ephemeral and has a very short lifecycle. It exists only to be returned from the method that performed the inventory operation. Its scope is typically confined to the immediate handling of that operation's result.
- **Destruction:** The object is managed by the Java Garbage Collector and is eligible for collection as soon as all references to it are dropped. No manual cleanup is required.

## Internal State & Concurrency
- **State:** Immutable. All fields are final and are set only once during construction. A ResourceSlotTransaction instance represents a snapshot of an outcome at a specific moment and can never be changed.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. An instance can be safely shared and read across multiple threads without any external synchronization.

    **Warning:** While this object is thread-safe, the inventory containers that generate it are **not**. All modifications to inventory state must be performed on the main server thread or be protected by appropriate locks to prevent data corruption.

## API Surface
The public contract is focused on retrieving the results of the transaction and translating its context between nested containers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | ResourceQuantity | O(1) | Returns the original resource and quantity requested in the operation. |
| getConsumed() | int | O(1) | Returns the actual quantity of the resource that was successfully added or removed. |
| getRemainder() | int | O(1) | Returns the quantity of the resource that could not be fulfilled by the operation. |
| toParent(...) | ResourceSlotTransaction | O(1) | Creates a new transaction with its slot index remapped to a parent container's coordinate space. |
| fromParent(...) | ResourceSlotTransaction | O(1) | Creates a new transaction with its slot index remapped from a parent to a local container's coordinate space. Returns null if the slot is out of bounds. |

## Integration Patterns

### Standard Usage
This object is not used to initiate actions; it is used to inspect the results. The caller of an inventory operation receives an instance and uses its state to determine the next steps in game logic.

```java
// A system requests to take 10 wood from an inventory container
ResourceQuantity request = new ResourceQuantity("hytale:wood", 10);
ResourceSlotTransaction result = container.takeResource(request);

if (result.succeeded()) {
    // Check how much was actually taken
    int amountTaken = result.getConsumed();
    log.info("Successfully consumed " + amountTaken + " wood.");

    if (result.getRemainder() > 0) {
        log.warn("Could not fulfill the entire request. " + result.getRemainder() + " wood remains needed.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using *new ResourceSlotTransaction()*. Doing so creates a synthetic result that does not reflect the actual state of any inventory, leading to desynchronization and logic errors. These objects must only be created by the authoritative inventory system.
- **Ignoring the Result:** Do not assume an inventory operation succeeded without inspecting the returned transaction. Always check the *succeeded*, *getConsumed*, and *getRemainder* fields to confirm the outcome.

## Data Pipeline
A ResourceSlotTransaction is a terminal object in a data flow, representing a finalized result. It is generated by the inventory system and consumed by higher-level game logic or network systems.

> Flow:
> Game Logic Request (e.g., CraftingSystem) -> InventoryManager.takeResource(query) -> ItemContainer.processTake() -> **new ResourceSlotTransaction()** -> Return to Game Logic -> (Optional) Network Packet Generation

