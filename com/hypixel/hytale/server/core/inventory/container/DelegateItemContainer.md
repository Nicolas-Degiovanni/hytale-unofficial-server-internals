---
description: Architectural reference for DelegateItemContainer
---

# DelegateItemContainer

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Decorator

## Definition
```java
// Signature
public class DelegateItemContainer<T extends ItemContainer> extends ItemContainer {
```

## Architecture & Concepts

The DelegateItemContainer is an implementation of the **Decorator** design pattern. Its primary architectural purpose is to augment an existing ItemContainer instance with additional functionality—specifically, a sophisticated filtering layer—without altering the original container's class.

It acts as a transparent wrapper, or proxy, around a "delegate" ItemContainer. For most operations, such as reading or writing item data, it forwards the request directly to the delegate. However, it intercepts all modification-check operations (e.g., `cantAddToSlot`, `cantRemoveFromSlot`) to enforce its own set of rules before consulting the delegate.

This design provides significant flexibility. Core inventory logic can operate on a standard ItemContainer, while UI-specific or gameplay-specific logic can wrap that same container with a DelegateItemContainer to enforce temporary or contextual rules, such as allowing only quest items to be placed in a specific slot.

The key features added by this decorator are:
1.  **Global Filtering:** A high-level switch to disallow all inputs or all outputs.
2.  **Per-Slot Filtering:** Granular rules that can be applied to individual slots for specific actions like adding, removing, or dropping items.
3.  **Event Propagation:** It intelligently intercepts and transforms events from the delegate container, ensuring that any system listening to the decorator receives events that correctly identify the decorator as the source, maintaining abstraction.

## Lifecycle & Ownership
-   **Creation:** A DelegateItemContainer is created manually by a system that needs to apply filtering rules to a pre-existing ItemContainer. It is not managed by a central registry and is instantiated via its public constructor, passing the container to be wrapped as an argument.

-   **Scope:** The lifecycle of a DelegateItemContainer is tightly coupled to the object it decorates and the context in which it was created. It is typically short-lived, existing only as long as the special filtering behavior is required (e.g., for the duration a player has a specific UI open).

-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as all references to it are released. There is no explicit `destroy` or `close` method.

## Internal State & Concurrency
-   **State:** The class is stateful. Its state consists of a reference to the delegate ItemContainer and the filter configuration, which includes the `globalFilter` and the `slotFilters` map. The actual inventory items are not stored in this class; they remain within the delegate.

-   **Thread Safety:** This class is designed to be thread-safe.
    -   Filter configuration methods (`setGlobalFilter`, `setSlotFilter`) are safe for concurrent use. The internal `slotFilters` map uses `ConcurrentHashMap` and `Int2ObjectConcurrentHashMap` to prevent race conditions during filter updates.
    -   Core inventory operations (`readAction`, `writeAction`) do not introduce any new locking. They forward directly to the delegate, relying entirely on the delegate's own concurrency control and locking mechanisms to ensure data integrity.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDelegate() | T | O(1) | Returns the underlying, wrapped ItemContainer instance. Use with extreme caution. |
| setGlobalFilter(filter) | void | O(1) | Applies a high-level filter to all slots, such as ALLOW_ALL or DENY_ALL. |
| setSlotFilter(action, slot, filter) | void | O(log N) | Sets a granular filter for a specific action on a single slot. Pass null to remove the filter. |
| registerChangeEvent(priority, consumer) | EventRegistration | O(1) | Registers a change listener. Correctly handles and transforms events originating from the delegate. |

## Integration Patterns

### Standard Usage
The standard pattern is to wrap an existing container, apply filters, and then expose only the wrapper to other systems. This encapsulates the filtering logic and prevents bypass.

```java
// 1. Get the base container (e.g., a player's hotbar)
ItemContainer playerHotbar = player.getInventory().getHotbarContainer();

// 2. Wrap it with the decorator
DelegateItemContainer<ItemContainer> filteredHotbar = new DelegateItemContainer<>(playerHotbar);

// 3. Apply filtering rules (e.g., only allow swords in the first slot)
SlotFilter swordFilter = (action, container, slot, item) -> item.isSword();
filteredHotbar.setSlotFilter(FilterActionType.ADD, (short) 0, swordFilter);

// 4. Use the filtered container for game logic
// This call will fail unless the item is a sword.
filteredHotbar.tryToAdd(new ItemStack("diamond_sword"));
```

### Anti-Patterns (Do NOT do this)
-   **Modifying the Delegate Directly:** Retrieving the delegate and modifying it directly will bypass all filtering logic defined in the DelegateItemContainer. This breaks the abstraction and can lead to an inconsistent state and subtle bugs.

    ```java
    // AVOID THIS
    DelegateItemContainer<ItemContainer> wrapper = ...;
    
    // This bypasses all of the wrapper's filters!
    wrapper.getDelegate().internal_setSlot((short) 0, new ItemStack("illegal_item"));
    ```

-   **Listening to Delegate Events:** Registering event listeners on the original delegate instead of the wrapper will result in receiving untransformed events. The event source will be the delegate, which may confuse systems that are only aware of the wrapper. Always register listeners on the outermost decorator.

## Data Pipeline

The primary data flow for this component involves the interception of modification checks. The flow ensures that the decorator's rules are evaluated before the underlying container's rules.

> Flow for an "add item" check:
>
> `Game System Call` -> `DelegateItemContainer.cantAddToSlot()` -> `Check this.globalFilter` -> `Check this.slotFilters` -> `delegate.cantAddToSlot()` -> `Return Combined Boolean Result`

