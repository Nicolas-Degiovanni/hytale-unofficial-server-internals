---
description: Architectural reference for BodyMotionWanderInCircle
---

# BodyMotionWanderInCircle

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionWanderInCircle extends BodyMotionWanderBase {
```

## Architecture & Concepts
The BodyMotionWanderInCircle class is a specific movement constraint component within the Hytale server's NPC AI framework. It does not generate movement goals itself; rather, it modifies or limits movement proposed by other systems, such as a pathfinder or a higher-level behavior node. Its primary function is to ensure an NPC remains within a defined circular or spherical boundary.

This component operates in two distinct modes for determining the boundary's center point:
1.  **Flocking Mode:** When the `flock` flag is enabled, the reference point for the circle is dynamically set to the current position of the NPC's flock leader. This allows for mobile, group-based patrol zones where the entire group is leashed to its leader.
2.  **Leash Mode:** When flocking is disabled, the reference point is a static `leashPoint` defined in the NPCEntity component. This is used for tethering an NPC to a fixed location, such as a guard post or a home point.

The core logic resides in the `constrainMove` method, which intercepts a proposed movement vector. It calculates whether the endpoint of this movement would fall outside the allowed radius. If so, it computes the intersection point with the boundary and returns a scaled-down movement distance, effectively stopping the NPC at the edge. The constraint can be a 2D circle projected onto the world's normal plane or a full 3D sphere.

## Lifecycle & Ownership
-   **Creation:** Instances of BodyMotionWanderInCircle are not created directly. They are instantiated by the server's asset loading system via a corresponding builder, `BuilderBodyMotionWanderInCircle`. This process typically occurs when an NPC's behavior tree or role is loaded from a configuration file.
-   **Scope:** The object's lifetime is tied to the NPC behavior that uses it. It is a stateful, per-NPC-instance component. A unique instance exists for each NPC that has this specific wandering behavior active. It is not shared.
-   **Destruction:** The instance is eligible for garbage collection when the owning NPC is unloaded or when its AI switches to a different behavior state that no longer uses this constraint.

## Internal State & Concurrency
-   **State:** This class is stateful.
    -   The configuration fields `radius`, `flock`, and `useSphere` are final and are set at creation time, making them immutable for the lifetime of the object.
    -   The `referencePoint` field is a mutable `Vector3d`. It is used as a temporary, reusable vector to store the calculated center of the wander area. This is an optimization to prevent object allocation during each game tick.

-   **Thread Safety:** This class is **not thread-safe**. The mutable `referencePoint` field makes concurrent access from multiple threads dangerous, as it would create a race condition. All interactions with an instance of this class must be synchronized or, more typically, confined to the single main server thread responsible for updating NPC logic.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| constrainMove(...) | double | O(1) | Overrides the base class method. Calculates the maximum allowed movement distance along a proposed path to keep the entity within the defined radius. Returns a scalar from 0.0 to `moveDist`. |
| getReferencePoint(...) | Vector3d | Amortized O(1) | **WARNING:** Returns a mutable, internal vector. Do not store or modify the returned reference. Determines the center of the wander area by querying ECS for either the flock leader's position or the NPC's leash point. |

## Integration Patterns

### Standard Usage
This component is not intended for direct invocation by game logic developers. It is configured within NPC asset files and managed automatically by the AI's `MotionController`. The controller calls `constrainMove` during the NPC's physics update phase, after a target velocity or position has been determined by other behaviors.

```java
// Conceptual example of how the MotionController would use this component.
// Developers do not write this code themselves.

// Get the proposed movement for this tick
Vector3d proposedMove = pathfinder.getProposedMove();
double moveDistance = proposedMove.length();

// Apply the wander-in-circle constraint
// 'wanderConstraint' is an instance of BodyMotionWanderInCircle
double allowedDistance = wanderConstraint.constrainMove(
    npcRef,
    npcRole,
    currentPosition,
    proposedMoveTarget,
    moveDistance,
    componentAccessor
);

// Update the final movement vector based on the constraint
Vector3d finalMove = proposedMove.normalize().multiply(allowedDistance);
transform.applyMovement(finalMove);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BodyMotionWanderInCircle()`. The object is complex and must be configured through its corresponding builder, which is handled by the asset pipeline.
-   **Instance Sharing:** Do not share a single instance of this class across multiple NPCs. The internal `referencePoint` state is specific to one NPC's context and will cause catastrophic bugs if shared.
-   **Concurrent Modification:** Do not call any methods on this object from an asynchronous task or a different thread. All NPC AI updates must occur on the main server tick thread.

## Data Pipeline
The flow of data through this component acts as a filter or a clamp on movement data.

> Flow:
> AI Behavior (e.g., Pathfinding) -> Proposed Target Position -> **BodyMotionWanderInCircle.constrainMove** -> Clamped Move Distance -> Physics System -> Final NPC Position Update

