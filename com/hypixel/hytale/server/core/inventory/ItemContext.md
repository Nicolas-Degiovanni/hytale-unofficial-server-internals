---
description: Architectural reference for ItemContext
---

# ItemContext

**Package:** com.hypixel.hytale.server.core.inventory
**Type:** Value Object / DTO

## Definition
```java
// Signature
public class ItemContext {
```

## Architecture & Concepts
The ItemContext class is a fundamental data structure within the server's inventory system. It is not a service or a manager; rather, it is an immutable data carrier that encapsulates the complete context of an item at a specific location. Its primary design purpose is to aggregate an ItemStack, the ItemContainer it resides in, and its specific slot index into a single, cohesive object.

This pattern significantly simplifies the design of higher-level inventory logic. Instead of passing multiple, disconnected parameters (container, slot, item) through method calls or event payloads, systems can pass a single, self-contained ItemContext. This improves code readability, reduces the surface area for errors, and ensures that any function operating on an item has all the necessary information about its origin and state at a specific moment in time.

It functions as a *snapshot*, representing the state of a specific inventory slot at the point of its creation.

### Lifecycle & Ownership
- **Creation:** ItemContext instances are created on-demand by inventory management systems. They are typically instantiated at the beginning of an inventory-related transaction, such as a player moving an item, a block being broken, or a command being executed. The creator is responsible for providing valid, non-null references to the container and item stack.

- **Scope:** The object is designed to be **short-lived and transient**. Its scope is typically confined to a single method call or the lifetime of a single event being processed. It is not intended to be cached or stored in long-term collections.

- **Destruction:** An ItemContext instance becomes eligible for garbage collection as soon as the operation that created it completes. No external systems maintain long-term references to it.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields (container, slot, itemStack) are declared as final and are set exclusively through the constructor. Once an ItemContext is created, its state cannot be altered. This is a critical design guarantee.

- **Thread Safety:** The class is **inherently thread-safe**. Its immutability ensures that it can be safely passed between threads without any risk of data corruption or race conditions. No external synchronization or locking is required when accessing an ItemContext instance.

## API Surface
The public API is minimal, consisting only of accessors for its constituent data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getContainer() | ItemContainer | O(1) | Returns the container where the item is located. |
| getSlot() | short | O(1) | Returns the numerical slot index within the container. |
| getItemStack() | ItemStack | O(1) | Returns the snapshot of the item stack itself. |

## Integration Patterns

### Standard Usage
ItemContext is used as a parameter object to provide full context to inventory-handling logic or event listeners.

```java
// A hypothetical service processing an inventory click
public void onPlayerInventoryClick(Player player, short clickedSlot) {
    ItemContainer inventory = player.getInventory();
    ItemStack stackInSlot = inventory.getItem(clickedSlot);

    // Do not proceed if the slot is empty
    if (stackInSlot.isEmpty()) {
        return;
    }

    // Create a context for this specific interaction
    ItemContext context = new ItemContext(inventory, clickedSlot, stackInSlot);

    // Pass the single context object to other systems
    ItemInteractionService.processInteraction(player, context);
    EventBus.post(new ItemUsedEvent(player, context));
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not store an ItemContext instance in a field or cache for later use. It represents a snapshot in time. The underlying ItemContainer may have changed since the context was created, making the ItemContext's data stale and unreliable. Always create a new ItemContext for each new, distinct operation.

- **Modification Assumption:** Never assume the ItemStack returned by getItemStack can be modified to affect the inventory. The ItemContext holds a reference, but the canonical state is in the ItemContainer. All inventory modifications must go through the ItemContainer's API.

- **Null Instantiation:** The constructor and its parameters are annotated as Nonnull. Attempting to create an ItemContext with a null container or item stack will result in a runtime exception and system instability.

## Data Pipeline
ItemContext serves as a temporary data packet within a larger inventory operation flow. It is created early in the process to bundle information and is consumed by downstream logic.

> Flow:
> Player Input -> Network Packet -> **InventoryManager (creates ItemContext)** -> EventBus -> Event Handlers (consume ItemContext) -> Inventory State Mutation

