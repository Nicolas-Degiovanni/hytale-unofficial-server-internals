---
description: Architectural reference for InternalContainerUtilItemStack
---

# InternalContainerUtilItemStack

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Utility

## Definition
```java
// Signature
public class InternalContainerUtilItemStack {
```

## Architecture & Concepts

The InternalContainerUtilItemStack class is a stateless utility that serves as the logical core for all item stack manipulations within the Hytale inventory system. It is not intended for direct use by feature developers; rather, it provides a set of `protected static` methods that are exclusively consumed by the ItemContainer class.

This class embodies the design principle of separating state from logic. The ItemContainer owns the state (the actual array or list of ItemStacks), while InternalContainerUtilItemStack encapsulates the complex, stateless algorithms for modifying that state. This separation makes the core inventory logic easier to test in isolation and keeps the ItemContainer class focused on state management, locking, and its public API contract.

Key architectural concepts include:

*   **Transactional Operations:** Every state-mutating method returns a detailed transaction object (e.g., ItemStackSlotTransaction, ItemStackTransaction). These objects provide a complete record of the operation, including success status, the state of items before and after, and any remaining items that could not be processed. This is critical for network synchronization and for higher-level systems to react to inventory changes reliably.
*   **Predictive Testing:** For many operations, there is a corresponding `test` method (e.g., testAddToExistingSlot). These methods perform a "dry run" of an operation, calculating the outcome without modifying the container's state. This allows the server to validate complex inventory actions or predict their results before committing to them, which is essential for UI feedback and preventing invalid client requests.
*   **Filtered Actions:** The boolean `filter` parameter present in most methods allows the calling ItemContainer to inject custom validation logic via its `cantAddToSlot` and `cantRemoveFromSlot` methods. This enables specialized containers (e.g., a furnace input slot) to enforce rules about what items are permissible without altering the core manipulation algorithms in this utility class.

## Lifecycle & Ownership

-   **Creation:** As a class containing only static methods, InternalContainerUtilItemStack is never instantiated. It is loaded into the JVM by the class loader when first referenced, typically by the ItemContainer class during server initialization.
-   **Scope:** The class and its methods are available for the entire application lifetime once loaded.
-   **Destruction:** The class is unloaded from the JVM when the server application shuts down. There is no instance-specific cleanup or lifecycle management required.

## Internal State & Concurrency

-   **State:** **Stateless**. This class holds no instance or static fields. All methods are pure functions with respect to the class itself; their output depends solely on the input arguments, primarily the provided ItemContainer and ItemStack. All mutable state is owned and managed by the ItemContainer instance passed into each method.

-   **Thread Safety:** The methods in this class are inherently thread-safe as they are stateless. However, the operations they perform on the supplied ItemContainer are **not** thread-safe by themselves. Concurrency control is the explicit responsibility of the calling ItemContainer. The common pattern observed in the source, `itemContainer.writeAction(() -> ... )`, indicates that the ItemContainer implements its own locking or transaction mechanism. This utility is designed to be executed safely *within* that protected context.

    **WARNING:** Calling any method in this class without the appropriate lock held by the ItemContainer will result in severe data corruption and race conditions.

## API Surface

The API consists entirely of `protected static` methods designed for internal use by ItemContainer. The following table highlights representative methods and their purpose.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| internal_addItemStack(container, itemStack, allOrNothing, fullStacks, filter) | ItemStackTransaction | O(N) | The primary implementation for adding an item stack to a container. It handles logic for filling existing stacks first, then empty slots. Returns a transaction detailing the result. |
| internal_setItemStackForSlot(container, slot, itemStack, filter) | ItemStackSlotTransaction | O(1) | Unconditionally sets or replaces the item stack in a specific slot. Bypasses stacking logic. |
| internal_removeItemStackFromSlot(container, slot, quantity, allOrNothing, filter) | ItemStackSlotTransaction | O(1) | Removes a specific quantity of an item from a single slot. |
| testAddToExistingSlot(container, slot, itemStack, maxStack, quantity, filter) | int | O(1) | Calculates how much of an item stack *would* remain after attempting to add it to a specific, existing stack, without actually modifying the container. |
| testRemoveItemStackFromItems(container, itemStack, quantity, filter) | int | O(N) | Calculates how much of a requested item removal *would* remain after scanning the entire container, without actually modifying it. |

*N = Capacity of the ItemContainer*

## Integration Patterns

### Standard Usage

This class should **never** be called directly from game systems, plugins, or any code outside of the `inventory.container` package. The correct pattern is to use the public API of an ItemContainer, which in turn delegates the core logic to this utility within a write-safe context.

```java
// DO NOT CALL InternalContainerUtilItemStack DIRECTLY.
// This example shows the intended INTERNAL call pattern within a hypothetical ItemContainer method.

public class ItemContainer {
    // ... other methods and fields

    public ItemStackTransaction addItem(ItemStack itemStack) {
        // The writeAction method is responsible for acquiring a lock.
        return this.writeAction(() -> {
            // The public method delegates the complex logic to the internal utility.
            return InternalContainerUtilItemStack.internal_addItemStack(
                this,
                itemStack,
                false, // allOrNothing
                false, // fullStacks
                true   // filter
            );
        });
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Invocation:** Calling any method of InternalContainerUtilItemStack from outside the ItemContainer class is a severe design violation. This bypasses validation, locking, and event handling that the ItemContainer is responsible for, leading to an inconsistent game state.
-   **Bypassing Write Actions:** Executing these methods on an ItemContainer without first acquiring the appropriate lock (e.g., by calling it outside of an `itemContainer.writeAction(...)` lambda) will break thread safety and cause unpredictable item duplication or loss in a multithreaded server environment.

## Data Pipeline

The methods in this class form a critical step in the server's inventory management data flow. They are the point where an abstract intention ("add this item") is converted into concrete data mutation and a verifiable result.

> Flow:
> High-Level API Call (e.g., PlayerInventory.addItem) -> ItemContainer Public Method -> ItemContainer Write Lock Acquisition -> **InternalContainerUtilItemStack** (Logic Execution) -> Raw Data Mutation in Container -> Return Transaction Result -> Network Packet Generation -> Client State Synchronization

