---
description: Architectural reference for ConnectedBlockPatternRule
---

# ConnectedBlockPatternRule

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Data Object / Configuration Model

## Definition
```java
// Signature
public class ConnectedBlockPatternRule {
```

## Architecture & Concepts
The **ConnectedBlockPatternRule** is a fundamental data structure that represents a single, declarative condition within the Connected Blocks system. It is not an active service or manager; rather, it is a passive data container that defines a specific requirement for a block's neighbor.

This class acts as a building block for defining complex auto-tiling and block-connection behaviors. For example, a fence post's appearance might change based on whether an adjacent block is also a fence post. A collection of **ConnectedBlockPatternRule** instances would be used to describe all such conditions.

Each rule specifies a relative position and a set of criteria that the block at that position must meet. These criteria can include block type, shape, placement orientation, or membership in a predefined list. The rule can be configured as either an inclusionary (**INCLUDE**) or exclusionary (**EXCLUDE**) condition.

The primary mechanism for creating instances of this class is through its static **CODEC** field, which deserializes rule definitions from asset files (e.g., JSON). This design separates the complex connection logic from the core game code, allowing designers to define and modify block behaviors without recompiling the engine.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale serialization framework using the static **CODEC** field. This process occurs when the server loads game assets, specifically those defining connectable blocks. Manual instantiation is an anti-pattern and will result in a non-functional object.

- **Scope:** An instance of **ConnectedBlockPatternRule** lives as long as its parent asset is held in memory. It is typically loaded once at server startup and persists for the entire session.

- **Destruction:** The object is eligible for garbage collection when the asset definition that contains it is unloaded from the asset management system.

## Internal State & Concurrency
- **State:** The object's state is **mutable only during deserialization** by its **CODEC**. Once constructed and loaded by the asset system, it should be treated as **effectively immutable**. All public methods are getters, enforcing a read-only contract for consumers. It holds configuration data and does not cache runtime state.

- **Thread Safety:** The class is **conditionally thread-safe**. The creation process via the codec is not guaranteed to be thread-safe and must be performed in a single-threaded context, such as the main asset loading thread. However, once an instance is fully constructed, it is safe to be read by multiple threads concurrently without external synchronization. This is critical for performance in multi-threaded world generation or block update systems.

## API Surface
The public API is composed entirely of simple accessors for retrieving rule parameters.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRelativePosition() | Vector3i | O(1) | Returns the block position to check, relative to the central block. |
| getBlockTypes() | HashSet<String> | O(1) | Returns the set of specific block type names this rule matches against. |
| getShapeBlockTypeKeys() | Set | O(1) | Returns a set of block entries, likely matching specific block states or shapes. |
| getFaceTags() | ConnectedBlockFaceTags | O(1) | Returns the conditions related to the faces of the target block. |
| getBlockTypeListAssets() | BlockTypeListAsset[] | O(1) | Returns an array of asset references that define pre-configured lists of block types. |
| getPlacementNormals() | AdjacentSide[] | O(1) | Returns the valid placement faces this rule applies to. Can be null. |
| isInclude() | boolean | O(1) | Returns true if this is an inclusionary rule (must match) or false for an exclusionary rule (must not match). |

## Integration Patterns

### Standard Usage
Direct interaction with this class is uncommon for most developers. It is primarily used internally by the Connected Blocks evaluation system. The following example illustrates a hypothetical evaluation loop.

```java
// A system checks if a rule matches the state of the world
public boolean evaluateRule(ConnectedBlockPatternRule rule, World world, Vector3i origin) {
    Vector3i targetPos = origin.add(rule.getRelativePosition());
    BlockState targetBlock = world.getBlockState(targetPos);
    
    // Simplified check: does the target block type exist in the rule's list?
    boolean typeMatch = rule.getBlockTypes().contains(targetBlock.getTypeId());
    
    // The rule's final result depends on whether it's an INCLUDE or EXCLUDE rule
    if (rule.isInclude()) {
        return typeMatch; // Must match
    } else {
        return !typeMatch; // Must NOT match
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use **new ConnectedBlockPatternRule()**. The object is complex and relies on the **CODEC** to be constructed correctly from a data source. Manual creation will lead to a default, empty, and useless rule.

- **State Mutation:** Do not attempt to modify the state of a **ConnectedBlockPatternRule** instance after it has been loaded. The internal collections are not designed for modification post-creation and doing so can lead to unpredictable behavior across the server.

## Data Pipeline
The data for a **ConnectedBlockPatternRule** flows from a configuration file on disk to an in-memory object used by the game engine for real-time evaluation.

> Flow:
> JSON Asset File -> Asset Manager -> **ConnectedBlockPatternRule.CODEC** -> **ConnectedBlockPatternRule Instance** -> Connected Block System -> World Block Update Evaluation

