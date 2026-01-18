---
description: Architectural reference for ItemContainerUtil
---

# ItemContainerUtil

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Utility

## Definition
```java
// Signature
public class ItemContainerUtil {
```

## Architecture & Concepts
The ItemContainerUtil class is a stateless utility designed to centralize and simplify the configuration of inventory slot behaviors. It acts as a configuration service for objects implementing the ItemContainer interface, particularly the SimpleItemContainer.

In the Hytale server architecture, inventories are not just passive data structures; they enforce complex rules about what items can be placed in which slots. For example, a helmet slot must only accept helmets. Instead of embedding this configuration logic within every system that creates an inventory (e.g., player creation, block placement), this utility provides a set of standard, reusable configuration "recipes".

This approach decouples the instantiation of an inventory from the application of its business logic, promoting consistency and reducing code duplication. It is the canonical source for defining standard inventory layouts like the player's equipment and armor slots.

### Lifecycle & Ownership
- **Creation:** As a static utility class, ItemContainerUtil is never instantiated. Its bytecode is loaded by the JVM ClassLoader upon the first static method invocation.
- **Scope:** The class and its static methods are available for the entire server session lifetime once loaded.
- **Destruction:** The class is unloaded when the server application terminates and the JVM shuts down.

## Internal State & Concurrency
- **State:** ItemContainerUtil is completely stateless. It contains no member fields and all its methods operate exclusively on the arguments provided. The behavior of its methods is deterministic.

- **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the ItemContainer instances passed to its methods are mutable and are often not thread-safe.

    **WARNING:** The caller is responsible for ensuring that no other thread is modifying an ItemContainer instance while it is being configured by this utility. Failure to provide proper synchronization will result in race conditions and unpredictable inventory behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| trySetArmorFilters(T container) | T | O(N) | Applies a standard set of player armor and equipment filters to a container. N is the container capacity. |
| trySetSlotFilters(T container, SlotFilter filter) | T | O(N) | Applies a single, uniform slot filter to every slot within a given container. N is the container capacity. |

## Integration Patterns

### Standard Usage
This utility should be invoked immediately after an inventory container is created to apply a standard ruleset. The methods return the container instance to allow for fluent-style chaining.

```java
// Example: Configuring a newly created player inventory
SimpleItemContainer playerInventory = new SimpleItemContainer(PLAYER_INVENTORY_CAPACITY);
ItemContainerUtil.trySetArmorFilters(playerInventory);
// The playerInventory is now configured with armor, accessory, and denied slots.

// Example: Configuring a simple chest that allows any item
SimpleItemContainer chestContainer = new SimpleItemContainer(CHEST_CAPACITY);
ItemContainerUtil.trySetSlotFilters(chestContainer, SlotFilter.ALLOW);
```

### Anti-Patterns (Do NOT do this)
- **Re-configuration:** Do not call these methods on an inventory that is already in use or contains items. Doing so will overwrite existing slot filters, which can lead to inconsistent state or invalid item placements. These methods are intended for initial setup only.
- **Ignoring Type Checks:** The methods internally check if the provided container is an instance of SimpleItemContainer. Passing a different implementation of ItemContainer will result in a silent no-op. This can be misleading, as the caller might assume configuration was successful when it was not.

## Data Pipeline
ItemContainerUtil does not participate in a continuous data pipeline. Instead, it acts as a one-time configuration step in the lifecycle of an inventory object.

> Flow:
> System requests new inventory -> `new SimpleItemContainer()` -> **ItemContainerUtil.trySetArmorFilters()** -> Configured inventory is returned -> Inventory is used by game systems (e.g., Player)

