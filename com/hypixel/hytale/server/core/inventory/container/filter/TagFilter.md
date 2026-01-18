---
description: Architectural reference for TagFilter
---

# TagFilter

**Package:** com.hypixel.hytale.server.core.inventory.container.filter
**Type:** Transient

## Definition
```java
// Signature
public class TagFilter implements ItemSlotFilter {
```

## Architecture & Concepts
The TagFilter is a concrete implementation of the **Strategy** pattern, conforming to the ItemSlotFilter interface. Its primary function is to act as a rule or predicate that determines whether a specific Item can be placed into an inventory slot.

This component is a cornerstone of Hytale's data-driven container system. It decouples the inventory slot's behavior from concrete item definitions. Instead of hard-coding a slot to accept a "Hytale Sword", a developer can configure it with a TagFilter for the "Weapon" tag. This allows any future item, from any source, to be placed in that slot provided it is tagged as a "Weapon", promoting extensibility and maintainability.

A TagFilter is configured with a single integer, the `tagIndex`, which is a unique identifier for a specific item property (e.g., "Tool", "Consumable", "Magic"). The filter's logic then simply checks for the presence of this tag on any given item.

## Lifecycle & Ownership
- **Creation:** TagFilter instances are created on-demand by higher-level systems responsible for container assembly, such as a ContainerFactory or during world generation when a chest with specific rules is placed. They are typically instantiated with a `tagIndex` read from an asset definition file.
- **Scope:** The lifetime of a TagFilter is bound to the ItemSlot or Container that references it. These are lightweight, short-lived objects. A server may create thousands of these instances during its operation.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup as soon as the owning slot or container is destroyed. No manual resource management is required.

## Internal State & Concurrency
- **State:** The TagFilter holds a single piece of state: the `tagIndex`. This state is **immutable**, as it is stored in a final field and set only once during construction. The object's behavior is therefore constant throughout its lifetime.
- **Thread Safety:** This class is inherently **thread-safe**. Its immutable state guarantees that it can be safely accessed and executed by multiple threads concurrently without any risk of race conditions or data corruption. No external locking or synchronization is necessary when using this filter.

## API Surface
The public contract is minimal, consisting of the constructor for initialization and the `test` method for evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TagFilter(int tagIndex) | constructor | O(1) | Creates a new filter instance bound to a specific tag index. |
| test(Item item) | boolean | O(1) | Evaluates if the provided Item has the configured tag. Returns true for null items (empty slots). |

## Integration Patterns

### Standard Usage
The TagFilter is intended to be instantiated and passed to an ItemSlot or other inventory component that requires filtering logic. The system then invokes the `test` method to validate item placement actions.

```java
// A system configures a slot to only accept items with tag index 12 (e.g., "Potions")
ItemSlotFilter potionFilter = new TagFilter(12);
ItemSlot potionSlot = new ItemSlot(potionFilter);

// During a player inventory action, the system validates the item
Item itemFromCursor = player.getHeldItem();
boolean canPlaceItem = potionSlot.getFilter().test(itemFromCursor);

if (canPlaceItem) {
    // Proceed with item transfer
}
```

### Anti-Patterns (Do NOT do this)
- **Re-use for Different Rules:** Do not attempt to modify a TagFilter instance to check for a different tag. Its state is immutable. If a different rule is needed, create a new TagFilter instance.
- **Invalid Tag Index:** Constructing a TagFilter with a `tagIndex` that does not correspond to a known tag in the game's asset registry will create a filter that may silently fail by always returning false for all items. This can lead to confusing and difficult-to-diagnose gameplay bugs.

## Data Pipeline
The TagFilter acts as a conditional gate within the server's inventory transaction pipeline. It does not transform data but rather validates it, allowing or halting the flow.

> Flow:
> Player Input (Inventory Move) -> Server InventoryManager -> **TagFilter.test(Item)** -> [Conditional: Allow/Deny] -> Container State Update -> Network Packet (Client Sync)

