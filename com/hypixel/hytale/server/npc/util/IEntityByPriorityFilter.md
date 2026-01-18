---
description: Architectural reference for IEntityByPriorityFilter
---

# IEntityByPriorityFilter

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface IEntityByPriorityFilter extends TriPredicate<Ref<EntityStore>, Ref<EntityStore>, ComponentAccessor<EntityStore>> {
```

## Architecture & Concepts
IEntityByPriorityFilter is a specialized, stateful strategy interface used within the server-side NPC AI framework. It defines a contract for filtering and selecting the single highest-priority target entity from a collection of potential candidates.

Architecturally, it elevates a standard functional predicate into a stateful, lifecycle-aware component. By extending TriPredicate, implementations can be seamlessly integrated into stream-based processing pipelines or other functional constructs that operate on entities. However, its primary role is not just to return a boolean, but to internally track the "best" entity it has evaluated so far during a filtering operation.

This component is central to NPC decision-making, particularly for targeting logic in combat or interactive behaviors. It allows different NPC roles to implement unique and complex targeting criteria (e.g., "target the closest player with the lowest health" or "target the entity with the 'Marked' debuff").

## Lifecycle & Ownership
- **Creation:** Implementations are instantiated on-demand by a higher-level system, typically an NPC's Role or AI behavior tree, at the beginning of a target acquisition tick.
- **Scope:** The lifecycle of an IEntityByPriorityFilter instance is extremely short and transient. It is designed to exist only for the duration of a single target-selection operation.
- **Destruction:** The caller that creates the filter instance is responsible for explicitly calling the `cleanup` method, preferably within a `finally` block. This is critical for releasing any cached references or resetting state, preventing memory leaks and logical errors in subsequent AI ticks.

## Internal State & Concurrency
- **State:** Implementations of this interface are fundamentally **mutable and stateful**. During the evaluation of multiple entities, the filter maintains an internal reference to the current highest-priority target discovered. This state is updated each time a new, higher-priority entity is found.
- **Thread Safety:** This interface and its implementations are **not thread-safe**. They are designed to be created, used, and destroyed within the confines of a single NPC's update tick. Sharing an instance across multiple threads or even across different AI ticks without re-initialization will result in severe race conditions and unpredictable targeting behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(Role) | void | O(1) | Initializes the filter with the context of the source NPC. **MUST** be called before use. |
| getHighestPriorityTarget() | Ref<EntityStore> | O(1) | Returns the highest-priority entity found during the filtering process. Returns null if no suitable target was found. |
| cleanup() | void | O(1) | Resets internal state and releases references. **MUST** be called after the operation is complete. |
| test(...) | boolean | O(1) | Inherited from TriPredicate. Evaluates a single entity, updates internal state if it is a better target, and returns the result. |

## Integration Patterns

### Standard Usage
The IEntityByPriorityFilter is intended to be used as a transient strategy object for a single targeting operation. The caller must strictly adhere to the init-use-cleanup lifecycle.

```java
// A concrete implementation is created for a single targeting check
SpecificTargetFilter filter = new SpecificTargetFilter();
try {
    // 1. Initialize with the context of the NPC performing the search
    filter.init(this.getNpcRole());

    // 2. Use the filter to process a stream of nearby entities
    // The .filter() call populates the internal state of the filter instance
    nearbyEntities.stream().filter(entityRef -> filter.test(entityRef, ...)).count();

    // 3. Retrieve the result from the filter's internal state
    Ref<EntityStore> bestTarget = filter.getHighestPriorityTarget();
    if (bestTarget != null) {
        // Act on the target
    }
} finally {
    // 4. CRITICAL: Always clean up to prevent resource leaks
    filter.cleanup();
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not cache and reuse a filter instance across multiple AI ticks. Its internal state will be invalid.
- **Omitting Cleanup:** Failure to call `cleanup` will lead to stale entity references being held, preventing garbage collection and potentially causing severe memory leaks.
- **Concurrent Access:** Never share a filter instance between threads. The internal state tracking the best target is not protected by locks and will be corrupted.
- **Skipping Initialization:** Calling `test` or `getHighestPriorityTarget` before `init` will result in a NullPointerException or undefined behavior, as the filter lacks the necessary context to make a decision.

## Data Pipeline
This component acts as a stateful processor in the NPC target acquisition pipeline.

> Flow:
> NPC AI Tick -> Gather Potential Targets (e.g., from a spatial query) -> Instantiate **IEntityByPriorityFilter** -> Stream of Entities -> Filter processes each entity, updating its internal "best target" -> `getHighestPriorityTarget()` is called -> AI System receives final target -> AI Action (e.g., Pathfinding, Attack)

