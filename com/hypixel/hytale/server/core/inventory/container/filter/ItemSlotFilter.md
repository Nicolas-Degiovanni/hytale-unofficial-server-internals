---
description: Architectural reference for ItemSlotFilter
---

# ItemSlotFilter

**Package:** com.hypixel.hytale.server.core.inventory.container.filter
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface ItemSlotFilter extends SlotFilter {
```

## Architecture & Concepts
The ItemSlotFilter interface is a specialized contract within the server's inventory framework, designed to enforce placement rules for items within container slots. It embodies the **Strategy Pattern**, allowing the generic ItemContainer to delegate slot-specific validation logic to a concrete implementation.

This interface simplifies the more general SlotFilter contract. Where SlotFilter provides the full context of an inventory action (the container, the slot index, the action type, and the full ItemStack), ItemSlotFilter abstracts this away to a single, fundamental question: "Is this specific *Item* type allowed here?".

This abstraction is critical for performance and clarity. Most inventory slot rules are based on the type of item (e.g., "only helmets go in the head slot", "only fuel goes in the furnace slot"), not on its quantity or metadata. By providing a default implementation that extracts the Item from the ItemStack, this interface allows developers to implement the most common validation logic with minimal boilerplate.

## Lifecycle & Ownership
As an interface, ItemSlotFilter itself has no lifecycle. The following applies to its concrete implementations.

-   **Creation:** Implementations are expected to be stateless and are typically instantiated once at server startup, often as singletons or flyweights. They are registered and associated with specific container slot definitions when the server loads game data.
-   **Scope:** An implementation's instance is designed to be application-scoped. A single instance, for example a theoretical OnlySwordsFilter, is shared across all containers and all players that require that specific rule.
-   **Destruction:** Instances are destroyed only upon server shutdown.

## Internal State & Concurrency
-   **State:** The contract is designed for **stateless** implementations. A filter should not contain any mutable fields or rely on instance-specific data. Its `test` method should be a pure function, where the output depends solely on the input arguments.
-   **Thread Safety:** When implemented correctly as stateless objects, filters are inherently **thread-safe**. The inventory system may invoke these filters from multiple threads processing different player actions concurrently. Any introduction of state into an implementation will break this guarantee and is a critical defect.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(actionType, container, slot, itemStack) | boolean | O(1) | **Default Method.** Translates a complex inventory action into a simple item-based test. Delegates to test(Item). |
| test(Item) | boolean | O(1) | **Abstract Method.** The core contract. Returns true if the provided Item is permitted in the slot governed by this filter. |

## Integration Patterns

### Standard Usage
A developer implements this interface to define a new slot rule. The implementation is then attached to a slot definition within a container's configuration. The system handles the invocation automatically.

```java
// 1. Define a concrete filter implementation
public class FuelSourceFilter implements ItemSlotFilter {
    @Override
    public boolean test(@Nullable Item item) {
        // Returns true only if the item has a fuel component
        return item != null && item.getComponent(FuelComponent.class) != null;
    }
}

// 2. In container configuration (conceptual)
// furnace.getSlot(FUEL_SLOT).setFilter(new FuelSourceFilter());
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Do not add fields to a filter implementation. The same instance is shared globally and must be re-entrant and thread-safe. Storing per-player or per-container state will cause severe concurrency bugs.
-   **Overriding the Default Method:** Avoid overriding the default `test(FilterActionType, ...)` method. Its purpose is to provide a stable, simplified abstraction layer. Bypassing it re-introduces complexity that this interface was designed to remove. Only override it if the validation logic depends on the specific action type (e.g., allowing removal but not addition), which is an exceptional case.

## Data Pipeline
ItemSlotFilter acts as a synchronous gate within the data flow of an inventory transaction. It does not transform data but rather validates it, halting the pipeline if its conditions are not met.

> Flow:
> Player Input -> Network Packet -> Server InventoryTransactionHandler -> **ItemSlotFilter.test(item)** -> [**Allowed**] -> ItemContainer State Change -> Network Sync Packet -> Client UI Update
>
> Flow (Rejected):
> Player Input -> Network Packet -> Server InventoryTransactionHandler -> **ItemSlotFilter.test(item)** -> [**Denied**] -> Transaction Rollback -> No State Change

