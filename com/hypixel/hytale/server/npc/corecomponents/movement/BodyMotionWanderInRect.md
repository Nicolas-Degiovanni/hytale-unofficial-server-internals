---
description: Architectural reference for BodyMotionWanderInRect
---

# BodyMotionWanderInRect

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionWanderInRect extends BodyMotionWanderBase {
```

## Architecture & Concepts
BodyMotionWanderInRect is a server-side NPC movement constraint component. It does not generate movement goals but rather enforces a bounding box on movement proposed by other AI systems. It specializes the generic BodyMotionWanderBase to confine an NPC to a predefined rectangular area centered on its leash point.

The core of its architecture is a spatial partitioning algorithm analogous to Cohen-Sutherland line clipping. The space around the NPC's leash point is divided into a 3x3 grid of nine sectors. The central sector is the valid movement area. The eight outer sectors are invalid.

When the AI system proposes a move, this component checks the start and end points of the movement vector against these sectors.
- If the move remains entirely within the central (valid) sector, it is permitted without modification.
- If the move starts inside but ends outside, the component calculates the precise intersection point with the boundary and scales the permitted movement distance to stop the NPC exactly at the edge.
- If a move starts and ends in the same invalid outer sector, it is only permitted if it moves the NPC *closer* to the valid area. Otherwise, the move is rejected (returns 0.0 distance).
- If a move crosses between different invalid sectors (e.g., from "top-left" to "bottom-left"), it is rejected, preventing the NPC from "corner cutting" around the exterior of its boundary.

This component is critical for designers to create patrol zones, guard posts, or pens for creatures without building physical barriers in the world.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's NPC asset pipeline via its corresponding builder, BuilderBodyMotionWanderInRect. This occurs when an NPC's behavioral profile is loaded and its components are constructed from asset definitions.
- **Scope:** The object's lifetime is bound to the NPC's configured behavioral profile (its Role). It is a stateless configuration object that persists as long as the NPC definition is loaded.
- **Destruction:** The object is eligible for garbage collection when the parent NPC definition is unloaded from memory. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** Immutable. The component's state, consisting of the rectangle's width and depth, is defined at construction via the builder and is stored in final fields. It holds no per-tick or per-NPC instance data.
- **Thread Safety:** This class is inherently thread-safe. Its immutable state and the fact that its core logic in constrainMove operates solely on method arguments make it safe for concurrent use by multiple NPC update threads without locks or synchronization.

## API Surface
The public contract is fulfilled by overriding a method from its parent class. Internal helper methods are protected.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| constrainMove(...) | double | O(1) | **Core Logic.** Calculates the maximum allowed travel distance along a proposed movement vector to keep the NPC within the defined rectangle. Returns a value from 0.0 to the original moveDist. |

## Integration Patterns

### Standard Usage
This component is not invoked directly. It is registered as part of an NPC's Role. The server's core AI movement loop retrieves this component and uses it as a filter or constraint when processing movement updates for the associated NPC.

```java
// Hypothetical usage within an NPC AI update loop
NPCEntity npc = ...;
BodyMotionWanderInRect constraint = npc.getRole().getMotionConstraint();
Vector3d currentPos = npc.getPosition();
Vector3d proposedTarget = npc.getAI().findNextWanderTarget();
double maxMoveDist = npc.getSpeed() * deltaTime;

// The constraint component scales down the movement if it would exit the boundary
double allowedMoveDist = constraint.constrainMove(
    npc.getRef(),
    npc.getRole(),
    currentPos,
    proposedTarget,
    maxMoveDist,
    entityStoreAccessor
);

// The final velocity is based on the allowed distance
Vector3d finalVelocity = proposedTarget.sub(currentPos).normalize().mul(allowedMoveDist);
npc.setVelocity(finalVelocity);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BodyMotionWanderInRect()`. The component must be configured and created through the `BuilderBodyMotionWanderInRect` as part of the asset loading system. Direct instantiation will bypass critical configuration steps.
- **State Misinterpretation:** Do not attempt to read the NPC's position from this component. It is a stateless calculator that only knows about the data passed into the `constrainMove` method for the duration of that call.

## Data Pipeline
This component acts as a clamp or filter in the NPC movement data pipeline. It takes a proposed movement and outputs a constrained movement distance.

> Flow:
> AI Behavior System (Proposes Target) -> Movement Vector -> **BodyMotionWanderInRect.constrainMove** -> Clipped Movement Distance -> Physics Engine (Applies final velocity) -> World State Update

