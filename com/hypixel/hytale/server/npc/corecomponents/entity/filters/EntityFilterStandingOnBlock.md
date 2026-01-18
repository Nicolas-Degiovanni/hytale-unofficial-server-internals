---
description: Architectural reference for EntityFilterStandingOnBlock
---

# EntityFilterStandingOnBlock

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterStandingOnBlock extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterStandingOnBlock is a specific, concrete implementation of the EntityFilterBase contract. Its sole purpose is to evaluate whether a given entity is currently standing on a block belonging to a predefined BlockSet. This component is a fundamental building block within the server-side NPC AI and behavior systems, used to construct conditional logic for actions, targeting, and state transitions.

Architecturally, this filter is notable for its dual-path evaluation logic, which represents a significant performance optimization:

1.  **Optimized NPC Path:** If the target entity is an NPC with an active Role, the filter delegates the check to the MotionController. This is the preferred path, as the MotionController likely caches ground-check information, providing a near-instantaneous result without requiring a direct world query.

2.  **General Entity Path:** If the target is not an NPC or lacks a MotionController, the filter falls back to a direct, more computationally intensive world lookup. It retrieves the entity's TransformComponent, calculates the block coordinates immediately below the entity's position, and queries the world's ChunkStore to identify the block ID. This ID is then checked against the configured BlockSet using the global BlockSetModule.

The static COST field (300) is a critical piece of metadata for the AI's query planner. It signals that this check is moderately expensive, encouraging the behavior system to evaluate cheaper filters first to short-circuit complex evaluations and maintain server performance.

### Lifecycle & Ownership
-   **Creation:** This object is not instantiated directly. It is created by the server's asset loading and NPC building pipeline. An instance of BuilderEntityFilterStandingOnBlock is populated from an asset definition (e.g., a JSON file for an NPC behavior), which is then used to construct this filter instance.
-   **Scope:** The lifetime of an EntityFilterStandingOnBlock instance is tied to the lifecycle of the specific NPC behavior or Role that defines it. It is a configuration object, not a long-lived service.
-   **Destruction:** The object is eligible for garbage collection as soon as the owning behavior or Role is unloaded or replaced.

## Internal State & Concurrency
-   **State:** This class is stateful but effectively immutable after construction. It holds a single final field, blockSet, which is an integer ID resolved from the builder during instantiation. Its internal state does not change for the duration of its lifetime.
-   **Thread Safety:** The class itself contains no synchronization primitives and does not modify its own state. However, the matchesEntity method reads from shared, mutable world state (EntityStore, ChunkStore). Therefore, its thread safety is entirely dependent on the calling context. It is **unsafe** to invoke matchesEntity from a thread that does not have exclusive or managed access to the world state for the target entity. The server's AI processing loop is expected to enforce a single-threaded or job-based execution model to guarantee safe access.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | Evaluates if the target entity is standing on a block from the configured set. Complexity assumes efficient component and chunk lookups. |
| cost() | int | O(1) | Returns the static computational cost (300) for this filter type. |

## Integration Patterns

### Standard Usage
This filter is not intended for direct invocation by most game logic. It is designed to be configured within NPC asset files and executed by a higher-level behavior system, such as a behavior tree or state machine.

```java
// Conceptual example of how a behavior tree node might use this filter.
// Note: This is a simplified representation.

// In an NPC's behavior asset:
// "condition": {
//   "type": "standingOnBlock",
//   "blockSet": "natural_stone"
// }

// In the behavior tree execution engine:
EntityFilterBase filter = loadFilterFromAsset(asset.condition);
boolean conditionMet = filter.matchesEntity(self, target, role, store);

if (conditionMet) {
    // Execute behavior...
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to construct this class with `new`. The constructor requires a builder and support context provided only by the internal asset pipeline. Incorrect instantiation will fail or lead to a misconfigured filter.
-   **Ignoring Cost:** In systems that evaluate multiple filters, do not iterate and execute them in an arbitrary order. The `cost()` method should be used to prioritize cheaper checks first to avoid unnecessary world lookups and maintain performance.

## Data Pipeline
The flow of data during the evaluation of `matchesEntity` follows one of two distinct paths depending on the target entity's type.

> **Path A: Optimized NPC Target**
>
> `Ref<EntityStore>` -> `NPCEntity` Component Lookup -> `Role` -> `MotionController` -> `standingOnBlockOfType()` -> **boolean**

> **Path B: General Entity Target**
>
> `Ref<EntityStore>` -> `TransformComponent` Lookup -> `Vector3d` Position -> World Query -> `ChunkStore` -> `BlockChunk` -> `getBlock()` at (x, y-1, z) -> `BlockSetModule.blockInSet()` -> **boolean**

