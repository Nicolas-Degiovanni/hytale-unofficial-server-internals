---
description: Architectural reference for NeighbourBlockTagsLocationCondition
---

# NeighbourBlockTagsLocationCondition

**Package:** com.hypixel.hytale.builtin.adventure.worldlocationcondition
**Type:** Transient

## Definition
```java
// Signature
public class NeighbourBlockTagsLocationCondition extends WorldLocationCondition {
```

## Architecture & Concepts
The NeighbourBlockTagsLocationCondition is a specific implementation of the WorldLocationCondition strategy. Its purpose is to act as a highly configurable predicate, answering a specific question about a location in the game world: **"Do the blocks adjacent to this coordinate possess a certain set of tags?"**

This component is entirely data-driven, configured through the engine's asset loading and `Codec` system. It is a fundamental building block for procedural generation, entity spawning, and quest logic. For example, it can be used to enforce rules such as:
*   A specific flower can only spawn next to a block tagged as *Water*.
*   A vine can only generate on the side of a block tagged as *Log* or *Stone*.
*   A treasure chest can only appear if it is supported by at least two blocks tagged as *DungeonBrick*.

Architecturally, it decouples complex world-state queries from the systems that need them. A spawner or generator does not need to know *how* to check for neighboring blocks; it simply holds a reference to a WorldLocationCondition and invokes its `test` method.

## Lifecycle & Ownership
- **Creation:** Instances are not created manually using the `new` keyword. They are deserialized and instantiated by the engine's `Codec` system when a parent asset (e.g., a spawner definition, a biome configuration) is loaded from a data file. The static `CODEC` field defines the exact schema for this deserialization.
- **Scope:** The object's lifetime is strictly bound to the parent asset that defines it. It is an immutable data-transfer object once loaded and persists as long as its parent configuration is held in memory.
- **Destruction:** The object is de-referenced and subsequently garbage collected when its parent asset is unloaded by the AssetManager. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state (tagPatternId, neighbourDirection, support) is populated once during deserialization and is **effectively immutable** for the object's entire lifecycle. This class performs read-only operations on the world.
- **Thread Safety:** This class is inherently thread-safe. The `test` method is a pure function relative to the object's internal state and can be called from any thread without synchronization. It safely queries world data using `getNonTickingChunk`, which is designed for concurrent, read-only access from systems like world generation that operate outside the main game loop.

## API Surface
The public contract is minimal, consisting almost entirely of the behavior inherited from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(World, int, int, int) | boolean | O(1) | Evaluates the condition at the given world coordinates. Returns true if the specified neighboring block(s) match the configured TagPattern. This operation is constant time as it performs a maximum of four block lookups. |

## Integration Patterns

### Standard Usage
This component is not used directly. Instead, a higher-level system retrieves a configured instance of its parent, WorldLocationCondition, from a loaded asset and invokes it polymorphically. The calling system is unaware of the specific condition type it is evaluating.

```java
// Example from a hypothetical procedural decoration system

// The condition is loaded from a JSON/HOCON asset file
WorldLocationCondition placementRule = decorationAsset.getPlacementCondition();

// The system iterates through potential locations and tests the rule
if (placementRule.test(world, x, y, z)) {
    // Condition passed, place the decoration
    world.setBlock(x, y, z, decorationBlock);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new NeighbourBlockTagsLocationCondition()`. This will create a useless, unconfigured object that will fail at runtime. All instances must be defined in data files and loaded via the engine's `Codec` system.
- **State Modification:** Do not attempt to modify the object's fields after it has been created. It is designed as an immutable data container. Modifying its state post-creation can lead to unpredictable behavior across the engine.

## Data Pipeline
The `test` method initiates a read-only data flow to determine its result. The process does not generate any persistent data or side effects.

> Flow:
> World Coordinates -> **NeighbourBlockTagsLocationCondition::test** -> World::getNonTickingChunk -> BlockAccessor::getBlock -> Block ID -> AssetMap::getAsset(BlockType) -> Block Tags -> AssetMap::getAsset(TagPattern) -> TagPattern::test -> Boolean Result

