---
description: Architectural reference for ResourceFilter
---

# ResourceFilter

**Package:** com.hypixel.hytale.server.core.inventory.container.filter
**Type:** Transient

## Definition
```java
// Signature
public class ResourceFilter implements ItemSlotFilter {
```

## Architecture & Concepts
The ResourceFilter is a concrete implementation of the **Strategy Pattern**, conforming to the ItemSlotFilter interface. Its primary role within the server architecture is to enforce type-based rules on inventory slots. It acts as a gatekeeper, ensuring that a slot can only accept items belonging to a specific, pre-configured resource category (e.g., ores, logs, fuel).

This class is a fundamental building block for creating specialized containers. For instance, a furnace's fuel slot would employ a ResourceFilter configured to only accept items with a burnable resource type. It decouples the container's logic from the specific rules of its slots, allowing for flexible and reusable container definitions. The filter's logic is determined entirely by the ResourceQuantity object provided during its construction.

### Lifecycle & Ownership
- **Creation:** A ResourceFilter is instantiated by a higher-level system responsible for constructing an inventory container, such as a FurnaceContainer or a custom storage block. It is created with a specific ResourceQuantity that defines its filtering behavior for its entire lifetime.
- **Scope:** The lifecycle of a ResourceFilter is strictly bound to the inventory slot or container it governs. It is a short-lived, scoped object that exists only as long as its parent container is active.
- **Destruction:** The object is managed by the Java Garbage Collector. When the parent container is destroyed (e.g., a player closes the inventory, a chest block is broken), the ResourceFilter becomes unreachable and is subsequently garbage collected. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The internal state, a single final ResourceQuantity field, is set at construction and cannot be modified thereafter. This design guarantees that a filter's behavior is consistent and predictable throughout its lifecycle.
- **Thread Safety:** This class is inherently thread-safe. Its immutability ensures that it can be safely accessed and used by multiple threads without the need for external synchronization or locks. The test method is a pure function, free of side effects.

## API Surface
The public contract is minimal, focusing exclusively on the filtering operation defined by the ItemSlotFilter interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Item item) | boolean | O(1) | Evaluates if the provided Item is compatible with the configured resource type. Returns true for null items, representing empty slots. |
| getResource() | ResourceQuantity | O(1) | Returns the underlying ResourceQuantity object that defines the filter's behavior. |

## Integration Patterns

### Standard Usage
A ResourceFilter is not typically used in isolation. It is passed during the construction or configuration of an inventory slot to define its behavior.

```java
// Example: Creating a slot that only accepts a specific type of ore
// This code would exist within a custom container's initialization logic.

// 1. Define the resource type we want to filter for.
// WARNING: ResourceQuantity must be obtained from the appropriate registry.
ResourceQuantity oreResource = ...; // Assume this is configured for ores

// 2. Create the filter with the desired resource rule.
ItemSlotFilter oreOnlyFilter = new ResourceFilter(oreResource);

// 3. Assign the filter to a new inventory slot.
// The container will now use this filter to validate any item movement into this slot.
InventorySlot oreSlot = new InventorySlot(inventory, slotIndex, x, y, oreOnlyFilter);
```

### Anti-Patterns (Do NOT do this)
- **Post-Construction Modification:** Attempting to change the filter's behavior after it has been created is an anti-pattern. Due to its immutable design, this is not possible. If a different filtering rule is required, a new ResourceFilter instance must be created.
- **Null ResourceQuantity:** Constructing a ResourceFilter with a null ResourceQuantity will result in a NullPointerException and server instability. The constructor does not perform null checks, assuming valid inputs from the container system.

## Data Pipeline
The ResourceFilter acts as a validation checkpoint in the server's item handling pipeline. It does not transform data but rather permits or denies an action based on its rules.

> Flow:
> Player Action (e.g., Click to move item) → Server Network Handler → InventoryTransactionManager → **ResourceFilter.test(item)** → [ACTION_ALLOWED / ACTION_DENIED] → State Update Packet to Client

