---
description: Architectural reference for BlockTagOrItemIdField
---

# BlockTagOrItemIdField

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Configuration Model / Data Transfer Object

## Definition
```java
// Signature
public class BlockTagOrItemIdField {
```

## Architecture & Concepts
The BlockTagOrItemIdField class is a specialized configuration model used within the Adventure Mode objective system. Its primary purpose is to represent a condition that can be satisfied by either a specific item ID (e.g., *hytale:stone*) or a category of items defined by a block tag (e.g., *hytale:ores*). This provides flexibility for content designers, allowing them to define objectives that target either a single entity or a group of related entities.

Architecturally, this class is designed for deserialization. It is not intended for manual instantiation in procedural code. The static **CODEC** field is the central feature, defining how an instance is constructed from a data source like a JSON file. The codec enforces a critical validation rule: one and only one of *BlockTag* or *ItemId* must be defined.

A key performance optimization is implemented in the codec's **afterDecode** hook. Upon loading, the string-based *blockTag* is immediately resolved into an integer-based *blockTagIndex* via the global AssetRegistry. This pre-computation ensures that all subsequent runtime checks using `isBlockTypeIncluded` are highly efficient, avoiding repeated string-to-index lookups during active gameplay.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **Codec** system during the loading and deserialization of adventure mode configuration assets. A game designer defines an objective in a configuration file, and the codec instantiates this class to represent a part of that definition.
- **Scope:** The object's lifetime is bound to its parent configuration object, typically an adventure mode task or objective. It persists in memory as long as the objective is active, which could be for the entire server session.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup once its parent configuration object is unloaded or dereferenced. There are no explicit destruction methods.

## Internal State & Concurrency
- **State:** The internal state is mutable only during the deserialization process managed by the **CODEC**. After the **afterDecode** hook completes, the object's state (*blockTag*, *itemId*, *blockTagIndex*) should be considered effectively immutable. Runtime logic should only read from this object.
- **Thread Safety:** This class is **not thread-safe** during its creation phase. The codec assumes it is operating in a single-threaded context, such as the main server thread during asset loading. However, once fully constructed and initialized, the object is safe for concurrent reads from multiple threads, as its internal state no longer changes. All public methods, except for `consumeItemStacks`, are read-only.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlockTagIndex() | int | O(1) | Returns the pre-computed integer index for the block tag. |
| getItemId() | String | O(1) | Returns the specific item ID string. |
| isBlockTypeIncluded(String) | boolean | O(1) | Checks if the provided block type key matches the configured tag or item ID. This is the primary runtime check method. |
| consumeItemStacks(ItemContainer, int) | void | O(N) | A command method that removes a specified quantity of matching items from a given ItemContainer. Complexity depends on container implementation. |

## Integration Patterns

### Standard Usage
This class is an internal component of the objective system and is not typically used directly. A higher-level system, such as a task processor, would retrieve an instance from a loaded objective configuration and use it to validate player actions.

```java
// Hypothetical usage within an objective processor
ObjectiveTask task = objective.getCurrentTask();
BlockTagOrItemIdField requiredBlock = task.getRequiredBlock();

// Event handler for a player breaking a block
void onPlayerBreakBlock(Player player, String blockId) {
    if (requiredBlock.isBlockTypeIncluded(blockId)) {
        // Player has made progress on the objective
        task.incrementProgress();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BlockTagOrItemIdField()`. This bypasses the critical **CODEC** logic, including validation and the `afterDecode` hook that populates the `blockTagIndex`. A manually created object will be in an invalid and non-functional state.
- **Post-Creation Mutation:** Do not modify the public or protected fields of this class after it has been loaded. Changing `blockTag` or `itemId` at runtime will cause a state mismatch with the `blockTagIndex`, leading to incorrect behavior in `isBlockTypeIncluded`.

## Data Pipeline
The flow for this component is driven by configuration loading, not real-time data processing.

> Flow:
> Adventure Mode Config File (JSON) -> Hytale Codec Deserializer -> **BlockTagOrItemIdField Instance** -> Objective/Task System -> Runtime Gameplay Logic (e.g., Event Listeners)

