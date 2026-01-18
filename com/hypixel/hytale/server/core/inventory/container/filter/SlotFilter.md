---
description: Architectural reference for SlotFilter
---

# SlotFilter

**Package:** com.hypixel.hytale.server.core.inventory.container.filter
**Type:** Functional Interface

## Definition
```java
// Signature
public interface SlotFilter {
   SlotFilter ALLOW = (actionType, container, slot, itemStack) -> true;
   SlotFilter DENY = (actionType, container, slot, itemStack) -> false;

   boolean test(FilterActionType var1, ItemContainer var2, short var3, @Nullable ItemStack var4);
}
```

## Architecture & Concepts
The SlotFilter interface defines a behavioral contract for validating item placement within an ItemContainer. It embodies the Strategy Pattern, decoupling the container's core logic from the specific rules governing each of its slots. This allows for the creation of complex and varied inventory behaviors without modifying the container implementation itself.

A SlotFilter acts as a gatekeeper. Before any item is moved into a slot, the container delegates the decision to the associated SlotFilter. The filter's implementation can then evaluate a set of conditions—such as the item type, the action being performed, or the current state of the container—to determine if the operation is permissible.

This component is fundamental to creating specialized inventories like crafting tables, furnaces, or equipment screens where certain slots must enforce strict rules (e.g., a helmet slot only accepting helmet-type items, or a furnace fuel slot only accepting burnable items).

The two provided static instances, ALLOW and DENY, serve as universal default implementations for slots with no special restrictions or for slots that are completely locked, respectively.

## Lifecycle & Ownership
As an interface, SlotFilter itself has no lifecycle. The lifecycle pertains to its concrete implementations.

-   **Creation:** Implementations are typically instantiated by the system that builds the inventory UI or the ItemContainer. For example, a FurnaceContainerFactory would create a specific SlotFilter for its fuel slot and another for its input slot. The static ALLOW and DENY instances are created once at class-loading time and are globally available.
-   **Scope:** The scope of a SlotFilter implementation is tightly bound to the ItemContainer or specific slots it governs. It lives as long as its owning container.
-   **Destruction:** Implementations are eligible for garbage collection when their owning ItemContainer is destroyed. The static instances persist for the entire application lifetime.

## Internal State & Concurrency
-   **State:** The interface contract is inherently stateless. Implementations should be designed as pure functions whose output depends solely on their inputs. The provided ALLOW and DENY implementations are stateless and immutable.
    -   **Warning:** Creating stateful SlotFilter implementations is a significant anti-pattern. A filter that relies on mutable internal state can produce inconsistent results and is not safe for reuse across different containers or threads.
-   **Thread Safety:** The interface itself makes no guarantees. However, due to the expectation of statelessness, implementations are expected to be unconditionally thread-safe. The responsibility for synchronized access to the ItemContainer and ItemStack, which are passed as arguments, lies with the calling system (the ItemContainer itself).

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(actionType, container, slot, itemStack) | boolean | O(1) | Evaluates if the given ItemStack can be placed in the specified slot. This is the core decision-making method. |
| ALLOW | SlotFilter | O(1) | A static singleton implementation that always returns true. |
| DENY | SlotFilter | O(1) | A static singleton implementation that always returns false. |

## Integration Patterns

### Standard Usage
A SlotFilter is not used directly by application-level code. It is provided to an ItemContainer, which then uses it internally to validate operations.

```java
// Example of a custom filter implementation
// This filter only allows items with a "fuel" component.
public class FuelSlotFilter implements SlotFilter {
    @Override
    public boolean test(FilterActionType actionType, ItemContainer container, short slot, @Nullable ItemStack itemStack) {
        if (itemStack == null) {
            return true; // Always allow removing items
        }
        return itemStack.hasComponent(FuelComponent.class);
    }
}

// In a container's initialization logic:
ItemContainer furnaceInventory = new ItemContainer(3);
furnaceInventory.setFilterForSlot(FUEL_SLOT_INDEX, new FuelSlotFilter());
furnaceInventory.setFilterForSlot(OUTPUT_SLOT_INDEX, SlotFilter.DENY); // Nothing can be manually placed here
```

### Anti-Patterns (Do NOT do this)
-   **Complex Side Effects:** A filter's `test` method should never modify the state of the container or the item stack. Its sole purpose is to return true or false. Modifying arguments within the method leads to unpredictable and difficult-to-debug behavior.
-   **Ignoring Action Type:** The FilterActionType parameter provides crucial context about *why* the check is being performed (e.g., player drag, hopper insertion). Failing to consider this can break automation mechanics or other game systems.

## Data Pipeline
SlotFilter acts as a conditional gate in a control flow rather than a step in a data transformation pipeline.

> Flow:
> User Input or Game Logic (e.g., Player drags item) -> ItemContainer::tryMoveItem -> **SlotFilter::test** -> [Decision: Allow/Deny] -> ItemContainer state is updated or remains unchanged<ctrl63>

