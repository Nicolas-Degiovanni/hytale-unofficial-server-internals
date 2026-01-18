---
description: Architectural reference for TagSlotTransaction
---

# TagSlotTransaction

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Transient

## Definition
```java
// Signature
public class TagSlotTransaction extends SlotTransaction {
```

## Architecture & Concepts
The TagSlotTransaction is a specialized, immutable data record that represents the outcome of an inventory operation involving item tags. It extends the base SlotTransaction, adding context specific to queries that match items by a category (the *tag*) rather than a specific item type. For example, a request to "add 50 wood logs" to an inventory might target any item with the *wood log* tag.

This class is a fundamental component of the server's inventory system. It acts as a detailed receipt, capturing not just the state change of a single inventory slot, but also the metadata of the tag-based query that produced it. The key fields, *query* and *remainder*, provide critical feedback to the calling system about the operation's success. The *query* stores the original amount requested, while the *remainder* indicates how many items could not be fulfilled.

Its primary role is to decouple the inventory manipulation logic from the systems that initiate those changes. Instead of returning simple booleans, the inventory system produces these rich transaction objects, allowing callers to precisely understand the results of complex operations.

### Lifecycle & Ownership
- **Creation:** TagSlotTransaction instances are created exclusively by the server's core inventory management services. They are instantiated as the result of a function call that attempts to add, remove, or move items based on a tag identifier, such as from an ItemContainer.
- **Scope:** The object's lifetime is ephemeral. It is typically created, returned up the call stack, inspected by the caller, and then immediately becomes eligible for garbage collection. It does not persist beyond the scope of the inventory operation that generated it.
- **Destruction:** Managed by the Java garbage collector. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The TagSlotTransaction is **strictly immutable**. All fields, including those inherited from SlotTransaction, are final and set only once during construction. This design guarantees that a transaction record cannot be altered after its creation, ensuring it remains an accurate historical fact of a completed operation.
- **Thread Safety:** As an immutable object, TagSlotTransaction is inherently **thread-safe**. It can be safely shared and read across multiple threads without locks or other synchronization primitives. This is critical in a server environment where inventory operations may be triggered by various concurrent systems (e.g., player actions, world events, administrative commands).

## API Surface
The public API is focused on data retrieval and coordinate space transformation for nested containers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | int | O(1) | Returns the total quantity of items requested in the original tag-based operation. |
| getRemainder() | int | O(1) | Returns the quantity of items from the query that could not be fulfilled. A value of 0 indicates complete success. |
| toParent(...) | TagSlotTransaction | O(1) | Translates the transaction's slot index from a child container's local space to its parent's coordinate space. Essential for composite inventories. |
| fromParent(...) | TagSlotTransaction | O(1) | Translates the transaction's slot index from a parent container's space to a child's local space. Returns null if the slot is not within the child's bounds. |

## Integration Patterns

### Standard Usage
A developer should never instantiate this class directly. Instead, they receive it as a result from a higher-level inventory API call and inspect its state to determine the outcome.

```java
// A system requests to add 64 items with the "LOG" tag to a container.
// The inventory system processes this and returns a transaction record.
TagSlotTransaction result = container.addItemByTag(ItemTags.LOG, 64);

if (result.succeeded()) {
    // The operation affected at least one slot.
    // Check if the entire request was fulfilled.
    if (result.getRemainder() > 0) {
        System.out.println("Could not add all items. " + result.getRemainder() + " logs remain.");
    } else {
        System.out.println("Successfully added all 64 logs.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new TagSlotTransaction(...)`. The inventory system is the sole authority for creating valid transaction records. Manually creating one can lead to an inconsistent and untruthful representation of the inventory state.
- **Ignoring Remainder:** A common error is to only check the `succeeded()` flag. A transaction can be considered successful even if it only partially completes the request. Always check `getRemainder()` to confirm if the entire quantity was processed.
- **Misinterpreting Coordinates:** When working with nested inventories, failing to use `toParent` or `fromParent` will result in incorrect slot indices and lead to manipulating the wrong items.

## Data Pipeline
TagSlotTransaction does not process data; it *is* the data. It represents the final output of a data flow within the inventory system.

> Flow:
> High-Level API Call (e.g., `container.addItemByTag`) -> Internal Inventory Logic & Slot Calculation -> **TagSlotTransaction Instantiation** -> Return to Caller -> Caller Inspects Result (e.g., `getRemainder()`)

