---
description: Architectural reference for ConnectedBlockFaceTags
---

# ConnectedBlockFaceTags

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Value Object

## Definition
```java
// Signature
public class ConnectedBlockFaceTags {
```

## Architecture & Concepts
The ConnectedBlockFaceTags class is a specialized data container that defines the connection rules for a block on a per-face basis. It is a core component of the "Connected Blocks" system, which allows block models and textures to adapt based on their neighbors.

This class acts as a data-driven rule set. For any given block, it answers the question: "Which types of blocks can my north face connect to?". The "types" are represented by arbitrary string tags. For example, a wooden fence post might have tags on its North, South, East, and West faces allowing it to connect to other blocks tagged as *fence_wood*.

Instances of this class are not created programmatically. Instead, they are deserialized from block asset definition files using the provided static CODEC. This design decouples the connection logic from the engine code, allowing designers and modders to define complex block behaviors entirely through data.

## Lifecycle & Ownership
- **Creation:** Instances are exclusively instantiated by the Hytale **Codec** system during the asset loading phase. The system reads a block definition file (e.g., a JSON file), finds the relevant section, and uses the static CODEC field to construct a ConnectedBlockFaceTags object. A static, pre-allocated EMPTY instance is provided for blocks with no connection rules, following the Null Object pattern.
- **Scope:** The lifetime of an instance is bound to the lifetime of its parent block definition. It is treated as immutable configuration data that persists in memory as long as the corresponding block type is registered and available in the game.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the game engine unloads the asset pack containing the parent block definition.

## Internal State & Concurrency
- **State:** The primary internal state is the `blockFaceTags` map, which maps a direction (Vector3i) to a set of string tags. This state is populated once during deserialization via the CODEC and is considered immutable thereafter. The class is designed as a read-only data structure post-construction.

- **Thread Safety:** This class is **not** thread-safe by enforcement but is safe by convention. It is designed for a "write-once, read-many" lifecycle. Concurrent reads from multiple threads (e.g., the rendering thread and a world-logic thread) are safe, provided the object is not modified after its initial creation by the asset loader.

    **WARNING:** The `getBlockFaceTags()` method returns a direct, mutable reference to the internal map. Modifying this map externally will violate the class's immutability contract and lead to unpredictable behavior and severe concurrency issues across the engine. This reference is exposed for performance reasons and must be treated as read-only.

## API Surface
The public API is designed for querying connection rules, not for mutation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| contains(direction, tag) | boolean | O(1) | Checks if a specific face is configured to connect to a given tag. This is the primary query method. |
| getBlockFaceTags(direction) | Set&lt;String&gt; | O(1) | Retrieves the complete set of connection tags for a given face. Returns an empty set if no tags are defined. |
| getDirections() | Set&lt;Vector3i&gt; | O(1) | Returns the set of all directions that have connection rules defined for this block. |

## Integration Patterns

### Standard Usage
This object is typically retrieved from a block's definition and queried by the world rendering or block update systems to determine the correct block state or model to use.

```java
// Assume 'block' is the instance of the block being checked
// Assume 'neighborDirection' is the direction of the adjacent block (e.g., Vector3i.NORTH)

// 1. Get the connection rules from the block's definition
ConnectedBlockFaceTags connectionRules = block.getDefinition().getConnectedBlockFaceTags();

// 2. Check if the face can connect to a neighbor with the "wood_plank" tag
boolean canConnect = connectionRules.contains(neighborDirection, "wood_plank");

if (canConnect) {
    // Update block state to use a connected model/texture
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ConnectedBlockFaceTags()`. The internal state will be empty and the object will be useless. These objects must be created via the asset loading and codec pipeline.
- **State Mutation:** Do not modify the collections returned by `getBlockFaceTags()` or `getDirections()`. This violates the immutability contract and will cause engine-wide instability.
- **Null Checks:** Do not check for null. Block definitions that lack connection tags will be assigned the static `ConnectedBlockFaceTags.EMPTY` instance, which can be safely queried.

## Data Pipeline
This class is a destination for data, not a processor within a pipeline. Its role is to represent deserialized configuration data in memory.

> Flow:
> Block Definition File (e.g., JSON) -> Hytale Asset Loader -> **Codec Deserializer** -> **ConnectedBlockFaceTags Instance** -> Queried by Block Update & Rendering Logic

