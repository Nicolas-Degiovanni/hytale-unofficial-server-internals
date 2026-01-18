---
description: Architectural reference for TestRemoveItemSlotResult
---

# TestRemoveItemSlotResult

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Transient

## Definition
```java
// Signature
public class TestRemoveItemSlotResult {
```

## Architecture & Concepts
The TestRemoveItemSlotResult is a **Result Object**, a specialized Data Transfer Object (DTO) that encapsulates the outcome of a simulated inventory operation. Its primary role is to answer the question: "If I were to remove X items of a certain type, which slots would they be taken from, and how many would fail to be removed?"

This class is fundamental to the server's transactional inventory system. It allows server logic to preview the result of an item removal *without actually modifying the inventory state*. This pre-computation, often called a "dry run", is a critical pattern for preventing partial failures and ensuring that complex inventory manipulations (like crafting or trading) can be fully satisfied before any items are committed to being moved or destroyed. It effectively decouples the *query* phase from the *mutation* phase of an inventory transaction.

## Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by inventory container logic, typically as the return value of a method like `testRemoveItems`. The constructor is seeded with the initial quantity of items that the operation failed to find.
-   **Scope:** Method-scoped. This object is designed to be short-lived. It is created, returned, inspected by the caller, and then immediately discarded. It should not be stored or passed between long-lived systems.
-   **Destruction:** Becomes eligible for garbage collection as soon as the calling method completes its logic based on the result. There are no external references or cleanup procedures required.

## Internal State & Concurrency
-   **State:** The internal state is **mutable** but should be treated as read-only by consumers. It consists of a map of slot indices to item counts (the `picked` map) and an integer representing the remaining quantity that could not be satisfied. The class does not cache data and holds no references to other engine services.
-   **Thread Safety:** This class is **not thread-safe**. The underlying map, Object2IntOpenHashMap, is unsynchronized. The object is designed for use within a single, synchronous operation on the server's main game thread.

    **WARNING:** Sharing an instance of TestRemoveItemSlotResult across multiple threads will lead to race conditions and undefined behavior. It must not be stored in a location accessible by other threads.

## API Surface
The public API is minimal, focusing on retrieving the results of the simulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasResult() | boolean | O(1) | Returns true if the simulation found at least one item to remove. |
| getPickedSlots() | Set<Short> | O(1) | Returns a set of all slot indices from which items would be removed. |

## Integration Patterns

### Standard Usage
The intended use is to call a "test" method on an inventory, inspect the returned TestRemoveItemSlotResult, and then proceed with the actual operation if the test was successful.

```java
// Hypothetical inventory service
Inventory inventory = player.getInventory();
int quantityToRemove = 10;

// 1. Perform a dry run of the removal operation
TestRemoveItemSlotResult result = inventory.testRemoveItems(Items.IRON_INGOT, quantityToRemove);

// 2. Check if the entire operation can be satisfied
if (result.quantityRemaining == 0) {
    // 3. If successful, commit the actual removal using the simulation's results
    inventory.commitRemove(result);
} else {
    // Handle the failure case, e.g., notify the player
    player.sendMessage("Not enough iron ingots.");
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TestRemoveItemSlotResult()`. An instance created this way is meaningless as it does not represent the outcome of any real inventory simulation. It must only be obtained from an authoritative inventory system.
-   **State Mutation:** Do not attempt to modify the internal state of a received result object. It is a report of a past event and should be treated as immutable by the consumer.
-   **Asynchronous Caching:** Do not store this object for later use in an asynchronous callback. The inventory state may have changed between the time the test was run and when the callback executes, invalidating the result.

## Data Pipeline
This class acts as a terminal output from a simulation process. It does not transform or forward data; it is the data itself.

> Flow:
> Inventory Transaction Logic -> `Inventory.testRemoveItems(item, count)` -> **TestRemoveItemSlotResult** (Creation) -> Calling Service (Inspection)

