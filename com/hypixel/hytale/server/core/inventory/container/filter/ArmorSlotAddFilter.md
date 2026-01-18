---
description: Architectural reference for ArmorSlotAddFilter
---

# ArmorSlotAddFilter

**Package:** com.hypixel.hytale.server.core.inventory.container.filter
**Type:** Utility / Policy Object

## Definition
```java
// Signature
public class ArmorSlotAddFilter implements ItemSlotFilter {
```

## Architecture & Concepts
The ArmorSlotAddFilter is a concrete implementation of the **Strategy** design pattern, conforming to the ItemSlotFilter interface. Its sole responsibility is to enforce server-side placement rules for specialized armor slots within an inventory container.

This class acts as a predicate or a gatekeeper. When the server processes an inventory transaction—such as a player attempting to place an item in their helmet slot—this filter is invoked to validate the action. It ensures that only items designated for a specific armor slot (e.g., HEAD, CHEST, LEGS, FEET) can be placed within it. This is a critical component for maintaining game state integrity and enforcing core gameplay mechanics.

By encapsulating this specific rule into a dedicated, immutable class, the inventory system remains modular and extensible. Other filters can be created for different types of slots (e.g., crafting ingredients, fuel) without modifying the core container logic.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level inventory container system, such as a PlayerInventoryContainer, during the container's own initialization. Each armor slot within the container will be configured with a new instance of this filter, parameterized with the appropriate ItemArmorSlot enum.
-   **Scope:** Transient. The object's lifetime is tightly coupled to the inventory slot it governs. It is a lightweight, short-lived object that exists only as part of a container's configuration.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the inventory container it belongs to is destroyed, and no explicit cleanup is required.

## Internal State & Concurrency
-   **State:** **Immutable**. The core state, the target ItemArmorSlot, is a final field set exclusively at construction time. Once an ArmorSlotAddFilter is created, its filtering logic is fixed for its entire lifetime.
-   **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a single instance can be safely accessed and executed by multiple threads without any risk of race conditions or data corruption. The test method is a pure function with no side effects.

## API Surface
The public contract is minimal and focused on its single responsibility as a predicate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Item item) | boolean | O(1) | The core validation method. Returns true if the item is null (allowing the slot to be emptied) or if the item is armor matching the filter's configured slot type. |
| getItemArmorSlot() | ItemArmorSlot | O(1) | Returns the specific armor slot type this filter is configured to validate against. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in general gameplay code. It is an internal component used by the inventory system to define the behavior of a slot. The following demonstrates its conceptual application within a container's setup.

```java
// Conceptual example within an inventory container's initialization
// This code would exist inside a class like PlayerInventoryContainer.

// Create a slot for the player's head equipment
ItemSlot headArmorSlot = new ItemSlot();

// Assign the filter to enforce that only helmets can be placed here
ItemSlotFilter helmetFilter = new ArmorSlotAddFilter(ItemArmorSlot.HEAD);
headArmorSlot.setFilter(helmetFilter);

// Later, when a player action is processed...
Item itemPlayerIsTryingToPlace = ...;
boolean isPlacementAllowed = headArmorSlot.getFilter().test(itemPlayerIsTryingToPlace);

if (!isPlacementAllowed) {
    // Reject the inventory transaction
}
```

### Anti-Patterns (Do NOT do this)
-   **Misuse for Non-Armor Slots:** Do not apply this filter to generic inventory slots. Doing so would incorrectly prevent any non-armor item from being placed, breaking expected inventory behavior.
-   **Attempting to Modify State:** Do not attempt to alter the filter's behavior after construction via reflection or other means. The system relies on its immutability for predictable and safe execution.

## Data Pipeline
The ArmorSlotAddFilter serves as a validation step within the server's inventory transaction pipeline. It does not transform data but rather permits or denies its flow.

> Flow:
> Player Input (Item Drag) -> Network Packet -> Server Inventory Transaction Handler -> **ArmorSlotAddFilter.test(item)** -> [Conditional] Transaction Commit/Reject -> Network Response -> Client UI Update

