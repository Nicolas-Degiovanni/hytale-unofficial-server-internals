---
description: Architectural reference for EntityPositionProvider
---

# EntityPositionProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Stateful Component

## Definition
```java
// Signature
public class EntityPositionProvider extends PositionProvider {
```

## Architecture & Concepts
The EntityPositionProvider is a fundamental component within the server-side NPC Artificial Intelligence framework, specifically serving the sensory systems. Its primary function is not merely to store a coordinate, but to maintain a *dynamic, self-validating reference* to a target entity.

This class acts as a smart pointer to an entity within the world's Entity-Component-System (ECS). Instead of holding a raw position that can become stale, it holds a `Ref<EntityStore>`, allowing it to query the entity's live data on demand.

The critical architectural feature is its built-in validation logic. On every query via `getTarget`, it implicitly checks two conditions:
1.  The underlying entity reference is still valid within the ECS.
2.  The target entity does not have a `DeathComponent`, meaning it is still alive.

If either check fails, the provider automatically clears its internal state, effectively forgetting the target. This self-cleaning behavior simplifies AI logic, as consumers of this class do not need to perform redundant liveness checks. It guarantees that any position data derived from it belongs to a valid, living entity.

## Lifecycle & Ownership
-   **Creation:** An EntityPositionProvider is instantiated on-demand by higher-level AI systems, such as a behavior tree or a sensor controller. It is typically created when an NPC needs to begin tracking a specific entity, for example, when a player enters an aggression radius.
-   **Scope:** The lifetime of an instance is tightly coupled to the AI behavior that requires it. It persists as long as the NPC needs to track its target. For instance, it may exist for the duration of a "chase" or "combat" state and be discarded when the NPC returns to an "idle" state.
-   **Destruction:** The object is eligible for garbage collection once the owning AI behavior or sensor system releases its reference. The `clear` method provides an explicit mechanism to sever the link to the target entity, effectively resetting the provider to its initial state without destroying the object itself.

## Internal State & Concurrency
-   **State:** This class is mutable. Its core state is the private `target` field, which holds a `Ref<EntityStore>`. This reference is modified by `setTarget` and can be nullified by `clear` or implicitly by a failed validation check within `getTarget`. It functions as a single-element cache for an entity reference.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal synchronization mechanisms like locks or atomic operations. It is designed to be created, modified, and read exclusively by the main server world thread that processes the game tick.

    **Warning:** Accessing an instance of EntityPositionProvider from any thread other than the primary world update thread will lead to severe concurrency issues, including race conditions and inconsistent state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setTarget(ref, accessor) | Ref<EntityStore> | O(1) | Establishes a tracked link to a target entity. This is the primary method for initializing the provider's state. |
| getTarget() | Ref<EntityStore> | O(1) | Returns the target reference if the entity is valid and alive. **Warning:** This method has side effects; it will clear the internal target if validation fails. |
| hasPosition() | boolean | O(1) | A convenience method to check if a valid, live target is currently being tracked. Internally calls `getTarget`. |
| clear() | void | O(1) | Manually invalidates the tracked target, releasing the internal reference. |

## Integration Patterns

### Standard Usage
The EntityPositionProvider is intended to be used as a stateful member of an AI agent's sensory or behavior component. The owning system sets a target in response to a world event and then polls the provider during its update tick to inform its decisions.

```java
// Conceptual example within an AI behavior's update method

// Assume this.positionProvider is an EntityPositionProvider instance
// and a target has been previously set.

public void onTick(AIContext context) {
    Ref<EntityStore> currentTarget = this.positionProvider.getTarget();

    if (currentTarget != null) {
        // Target is valid and alive, proceed with logic
        Vec3d targetPos = getPositionFromEntity(currentTarget);
        context.getPathfinder().setGoal(targetPos);
    } else {
        // Target is dead or invalid; the provider has cleared itself.
        // The AI should now transition to a new state, like "searching" or "idle".
        context.getBehaviorController().setState(AIState.IDLE);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Result Caching:** Do not call `getTarget` once and cache the returned `Ref` for use across multiple game ticks. The core value of this class is its per-tick validation. Always call `getTarget` when you need the most current, validated reference.
-   **Asynchronous Access:** Do not share an instance of this class with background threads or asynchronous tasks. All interactions must be synchronized with the main server game loop.
-   **State Assumption:** Do not call other methods that rely on the target's position without first checking the result of `getTarget` or `hasPosition` in the same tick. The target can be invalidated at any moment.

## Data Pipeline
The EntityPositionProvider acts as a source of validated positional data for other AI systems. It does not process data itself but rather gates access to it based on liveness and validity checks.

> Flow:
> World Event (e.g., Player Enters Zone) -> AI Sensor System -> **EntityPositionProvider.setTarget(entityRef)** -> AI Behavior Tick -> **EntityPositionProvider.getTarget()** -> Pathfinding or Combat System

