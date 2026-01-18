---
description: Architectural reference for EntityFilterInsideBlock
---

# EntityFilterInsideBlock

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterInsideBlock extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterInsideBlock is a specialized spatial query component within the server-side NPC AI framework. It serves as a concrete implementation of the *Strategy* pattern, providing a specific method for filtering entities based on their physical intersection with world geometry.

Its primary function is to answer a boolean question: **Is any part of a target entity's bounding box currently occupying a block that belongs to a predefined set?**

This class acts as a bridge between the high-level, abstract AI decision-making layer (e.g., "Is the player in water?") and the low-level, granular world data representation (Chunk Stores and Block Sections). It achieves this by iterating through the discrete block coordinates that an entity's bounding box overlaps and performing efficient lookups against the world state. The use of a BlockSet allows for flexible and data-driven definitions of what constitutes "inside", such as a set of all water blocks or all foliage blocks.

The component is heavily optimized for performance during its core operation by caching chunk and section lookups within the scope of a single `matchesEntity` call.

### Lifecycle & Ownership
- **Creation:** EntityFilterInsideBlock instances are not created directly via their constructor in gameplay code. They are instantiated by the NPC asset pipeline, configured through a corresponding `BuilderEntityFilterInsideBlock` and hydrated with a `BuilderSupport` context during server startup or asset loading.
- **Scope:** The lifetime of an instance is tied to the NPC behavior or `Role` that defines it. It persists as part of an NPC's configured component set and is reused for every evaluation of the filter condition.
- **Destruction:** The object is eligible for garbage collection when the parent NPC definition is unloaded from memory. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** This component is highly stateful, but its state is transient and scoped exclusively to a single invocation of the `matchesEntity` method. Fields such as `chunkStore`, `chunkIndex`, `blockChunk`, `chunkSection`, and the `matches` result are mutated during the execution of a check. This state is explicitly cleared at the end of the method call to ensure subsequent invocations are not contaminated. This pattern avoids repeated object allocations for performance-critical code.

- **Thread Safety:** **This class is not thread-safe.** Concurrent calls to `matchesEntity` on the same instance will result in a race condition, leading to corrupted internal state and unpredictable, incorrect results. All invocations must be synchronized externally or, as is standard, confined to the single thread responsible for ticking the NPC's AI.

## API Surface
The public contract is minimal, focusing on the evaluation and its associated performance cost.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(V) | Evaluates if the target entity's bounding box intersects with any block in the configured BlockSet. V is the volume of the bounding box in blocks. Throws NullPointerException if arguments are null. |
| cost() | int | O(1) | Returns the static, predefined computational cost of this filter. Used by the AI scheduler to weigh decisions. |

## Integration Patterns

### Standard Usage
This filter is not intended for direct invocation by general-purpose game logic. It is automatically invoked by the NPC's behavior tree or state machine when evaluating conditions for target selection or state transitions.

```java
// Conceptual example within an AI evaluation loop
// The filter is pre-configured on the npcRole object.

EntityFilterInsideBlock waterFilter = npcRole.getMovementFilter();
boolean isTargetInWater = waterFilter.matchesEntity(npcRef, targetRef, npcRole, entityStore);

if (isTargetInWater) {
    // Execute swimming behavior
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new EntityFilterInsideBlock()`. The `blockSet` field is critical and must be injected by the asset loading system via its builder. Direct instantiation will result in a non-functional component.
- **Concurrent Access:** Do not share an instance of this filter across multiple threads. The internal state is not protected by locks and will be corrupted, leading to severe and difficult-to-diagnose bugs.
- **State Inspection:** Do not attempt to read the internal state fields (e.g., `chunkIndex`, `blockChunk`) after a `matchesEntity` call. This state is considered an implementation detail and is reset at the end of the method, rendering it meaningless to external callers.

## Data Pipeline
The data flow for a single evaluation is a highly optimized, read-only query against the world state.

> Flow:
> AI System Request -> **EntityFilterInsideBlock.matchesEntity()** -> Get Target BoundingBox & Transform -> Iterate Overlapping Block Coordinates -> **(Internal Loop)** -> Get Chunk/Section (cached) -> Get Block ID -> Query BlockSetModule -> Update `matches` flag -> Return boolean result

