---
description: Architectural reference for ItemDropContainer
---

# ItemDropContainer

**Package:** com.hypixel.hytale.server.core.asset.type.item.config.container
**Type:** Polymorphic Configuration Model

## Definition
```java
// Signature
public abstract class ItemDropContainer implements IWeightedElement {
```

## Architecture & Concepts

The ItemDropContainer is the abstract base class for all item drop configurations within the server's loot system. It establishes a common contract for components that generate lists of potential item drops based on predefined rules and probabilities. This class and its concrete implementations form the backbone of Hytale's data-driven loot table architecture.

Architecturally, this system employs a combination of the **Strategy** and **Factory** patterns.
*   **Strategy Pattern:** Each concrete subclass (e.g., SingleItemDropContainer, ChoiceItemDropContainer) implements a different strategy for drop generation. One might drop a single item, another might choose one from a weighted list, and another might drop all items in its list.
*   **Factory Pattern:** The static `CODEC` field, a CodecMapCodec, acts as a factory. It deserializes structured data from asset files (e.g., JSON) into the appropriate concrete ItemDropContainer object based on a "Type" identifier. This powerful abstraction allows game designers to define complex, nested loot behaviors in data files without requiring changes to the core engine code.

This class is the central node for resolving what items are generated when a game event, such as a monster's death or a block being broken, triggers a loot drop.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the `new` keyword. They are instantiated exclusively by the Hytale `Codec` serialization system during the server's asset loading phase. The static `CODEC` registry maps a type name from a configuration file to a specific subclass constructor.
-   **Scope:** The lifetime of an ItemDropContainer object is bound to the parent asset that defines it (e.g., a monster definition, a block loot table). These objects are effectively immutable after being loaded and persist in memory for the entire server session, or until assets are reloaded.
-   **Destruction:** Objects are managed by the Java garbage collector. They are eligible for destruction when the parent asset configuration is unloaded from memory. There are no explicit cleanup or `close` methods.

## Internal State & Concurrency
-   **State:** The primary state is the `weight` field, which is used by parent containers to calculate selection probability. While the field is not declared `final`, instances of this class should be treated as **effectively immutable** after deserialization. Modifying state at runtime is an anti-pattern and will lead to unpredictable behavior.
-   **Thread Safety:** This class is **not thread-safe** for mutation. However, as it is intended to be used as read-only configuration data, it can be safely accessed by multiple threads for drop generation. The `populateDrops` method mutates the `List` passed into it, which is not synchronized. Callers are responsible for ensuring that concurrent drop calculations do not write to a shared list, or that access to such a list is properly synchronized.

## API Surface

The public API is focused on the single responsibility of generating a list of drops.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| populateDrops(drops, chanceProvider, droplistId) | void | Varies | The primary entry point for resolving drops. Populates the provided list based on the container's internal rules. Includes cycle detection for nested droplists. |
| getAllDrops(list) | List<ItemDrop> | Varies | Retrieves all *possible* drops from this container and its children, ignoring weights and probabilities. Primarily for tooling or UI display. |
| getWeight() | double | O(1) | Returns the configured weight of this container. Used by parent containers to determine selection chance. |

## Integration Patterns

### Standard Usage

Developers should never instantiate or manage ItemDropContainer objects directly. Interaction occurs through a higher-level system, such as a loot service, which retrieves a pre-configured loot table and uses it to generate drops.

```java
// Example: A high-level service using a pre-loaded loot table
LootTable monsterLoot = LootTableRegistry.get("ghoul.json");
List<ItemDrop> generatedDrops = new ArrayList<>();

// The service invokes populateDrops on the root container
monsterLoot.getRootContainer().populateDrops(generatedDrops, world.getRandom(), "ghoul_master_list");

// Process the generated drops
for (ItemDrop drop : generatedDrops) {
    world.spawnItem(drop.createItemStack(), location);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SingleItemDropContainer()`. The entire system relies on the `Codec` factory to build these objects from data files. Manual creation bypasses this and will result in unconfigured, non-functional objects.
-   **Runtime State Mutation:** Avoid modifying the `weight` field or any other internal state after an object has been loaded. This can have global consequences for drop rates and is not thread-safe.
-   **Ignoring Cycle Detection:** The `populateDrops` method includes a mechanism to prevent infinite recursion from misconfigured droplists (e.g., A references B, and B references A). While the system protects against a crash, creating such cycles in data files is a design flaw that should be avoided.

## Data Pipeline

The ItemDropContainer is a critical transformation step in the loot generation pipeline, converting static configuration data into a concrete list of potential items.

> Flow:
> Asset File (JSON) -> Server Asset Loader -> **CodecMapCodec Factory** -> **ItemDropContainer Instance** -> Loot Service Request -> `populateDrops` Execution -> Final `List<ItemDrop>`

---
## ItemDropContainer Subclasses

The power of the ItemDropContainer system comes from its concrete implementations, which are registered in the static initializer block.

### MultipleItemDropContainer
Drops **all** of the items or containers held within its internal list. It is used for guaranteed drops.

### ChoiceItemDropContainer
Selects **one** item or container from its internal list based on their relative weights. This is the primary mechanism for creating traditional, probability-based loot tables (e.g., 80% chance for junk, 15% for a common item, 5% for a rare item).

### SingleItemDropContainer
A simple container that holds and drops a single, specific ItemDrop.

### DroplistItemDropContainer
A reference container that does not hold items directly. Instead, it holds an identifier that points to another globally defined droplist asset. This allows for the creation of reusable, shared loot tables.

### EmptyItemDropContainer
A null-object pattern implementation. It contains no items and does nothing when `populateDrops` is called. It is useful for defining entities that are guaranteed to drop nothing.

