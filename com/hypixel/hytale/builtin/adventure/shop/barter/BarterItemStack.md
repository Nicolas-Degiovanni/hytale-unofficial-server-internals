---
description: Architectural reference for BarterItemStack
---

# BarterItemStack

**Package:** com.hypixel.hytale.builtin.adventure.shop.barter
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class BarterItemStack {
```

## Architecture & Concepts
The BarterItemStack class is a fundamental data structure representing a specific quantity of a single item type within the game's adventure mode systems, particularly for NPC shops and bartering transactions. It is not a service or manager, but rather a lightweight, passive data carrier.

Its primary architectural significance lies in its tight integration with Hytale's serialization framework, `com.hypixel.hytale.codec`. The static `CODEC` field exposes a complete definition for how to serialize, deserialize, and validate instances of this class. This pattern is central to Hytale's data-driven design, allowing game designers to define complex shop inventories, trade requirements, and loot tables in external configuration files (e.g., JSON) which are then safely loaded into the engine as strongly-typed BarterItemStack objects at runtime.

This class acts as the in-memory representation of an item stack defined in game data, bridging the gap between raw configuration and the live game logic.

### Lifecycle & Ownership
- **Creation:** Instances are created in two primary ways:
    1. **Declaratively (Most Common):** The Hytale engine's `Codec` system instantiates BarterItemStack objects when parsing game data files. The `BarterItemStack.CODEC` is used to decode the data into a valid object.
    2. **Programmatically:** Game logic can instantiate the class directly using `new BarterItemStack(...)` for dynamic scenarios, such as generating quest rewards or processing crafting results.
- **Scope:** BarterItemStack objects are short-lived and transient. Their lifetime is typically bound to a specific operation, such as the evaluation of a single trade, the rendering of a shop UI, or the granting of a reward. They are not persisted across sessions.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods. Once an object is no longer referenced by any system, it becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** The state is **mutable** but intended to be treated as immutable after creation. It consists of an `itemId` and a `quantity`. The internal fields are not final to support the `Codec` framework's instantiation process, which populates a default-constructed object.
- **Thread Safety:** This class is **not thread-safe**. As a simple DTO, it contains no internal synchronization mechanisms.

    **WARNING:** Sharing and modifying a BarterItemStack instance across multiple threads without external locking will lead to race conditions and unpredictable behavior. Instances should be confined to a single thread (e.g., the main game thread) or passed between threads as immutable data.

## API Surface
The public API is minimal, focusing on data construction and retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BarterItemStack(String, int) | constructor | O(1) | Creates a new instance of an item stack. |
| getItemId() | String | O(1) | Returns the unique identifier for the item. |
| getQuantity() | int | O(1) | Returns the number of items in the stack. |

## Integration Patterns

### Standard Usage
The most common pattern is defining item stacks within larger data files, which are then loaded by the engine. For programmatic use, direct construction is appropriate.

```java
// Example: Programmatically creating a reward for a quest
// Note: In a real scenario, "hytale:wood_plank" would likely be a constant.
BarterItemStack reward = new BarterItemStack("hytale:wood_plank", 10);
player.getInventory().addItemStack(reward);
```

### Anti-Patterns (Do NOT do this)
- **Invalid State Construction:** Do not programmatically create a BarterItemStack with a null `itemId` or a `quantity` less than 1. The `CODEC` enforces these constraints during data loading (`Validators.nonNull`, `Validators.greaterThanOrEqual(1)`), and runtime logic must uphold the same data integrity contract.
- **Post-Creation Mutation:** Avoid modifying a BarterItemStack after it has been passed to another system. While the class is technically mutable, it should be treated as an immutable value object to prevent side effects. For example, do not retrieve an item stack from an inventory, change its quantity, and expect the inventory to be updated. Instead, use the inventory's API to request a modification.

## Data Pipeline
BarterItemStack serves as a critical link in the data pipeline that transforms static game configuration into live game objects.

> Flow:
> JSON/HOCON File on Disk -> Asset Loading System -> **BarterItemStack.CODEC** -> **BarterItemStack Instance** -> Shop/Barter System Logic -> Player Inventory Update

