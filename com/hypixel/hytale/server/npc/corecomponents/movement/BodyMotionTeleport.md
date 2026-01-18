---
description: Architectural reference for BodyMotionTeleport
---

# BodyMotionTeleport

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionTeleport extends BodyMotionBase {
```

## Architecture & Concepts
BodyMotionTeleport is a specialized movement strategy component within the server-side NPC AI framework. It does not produce continuous steering forces like pathfinding or seeking behaviors. Instead, its sole purpose is to calculate and trigger an instantaneous, short-range teleport for an NPC.

Architecturally, this class functions as a **Command Generator** within the Entity-Component-System (ECS). Its primary method, computeSteering, evaluates conditions and, upon success, adds a `Teleport` component to its parent entity. The actual execution of the teleport is handled by a separate, dedicated system that processes `Teleport` components during the game tick, ensuring a clean separation of concerns between AI decision-making and physics simulation.

This component is designed for creating evasive or unpredictable movement patterns, such as an Enderman-style "blink" or a mage's "phase shift". It operates on a try-and-fail basis, attempting to find a valid destination within a configured radius and sector around a target. To prevent performance degradation from repeated failed attempts in complex environments, it is limited by a maximum number of tries (`MAX_TRIES`) and a cooldown period.

## Lifecycle & Ownership
-   **Creation:** BodyMotionTeleport instances are not created directly via their constructor. They are instantiated by the `BuilderBodyMotionTeleport`, which is typically populated from server-side asset definitions (e.g., JSON or HOCON files) that define an NPC's behavioral profile. This allows designers to configure teleportation behavior without modifying game code.
-   **Scope:** The lifetime of a BodyMotionTeleport object is tied to the `Role` or behavior state that owns it. It is activated when an NPC's AI enters a state requiring teleportation and remains dormant or is replaced otherwise. It is not a persistent, session-long object.
-   **Destruction:** The object is eligible for garbage collection once the owning `Role` is deactivated and no other references to it exist. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains internal state for its operational logic, including `tries` (remaining attempts), `cooldown` (time until next teleport is allowed), and vectors like `target` and `lastTriedTarget`. This state is critical for its iterative, cooldown-based behavior.

-   **Thread Safety:** **WARNING:** This class is not thread-safe and must not be accessed concurrently. It is designed to be operated exclusively by the single thread responsible for ticking the associated NPC's world or region. All interactions, especially calls to `computeSteering`, must be synchronized at a higher level by the core game loop to prevent race conditions when accessing entity components.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(...) | void | O(1) | Resets the internal state, primarily the `tries` counter. Called when the behavior becomes active. |
| computeSteering(...) | boolean | O(C) | Core logic. Attempts to find a valid teleport location. If successful, adds a `Teleport` component to the entity and returns. Complexity is dominated by the collision checks within `translateToAccessiblePosition`. |

## Integration Patterns

### Standard Usage
This component is not intended for direct use by game logic or scripting. It is an internal piece of the NPC AI system, driven by a parent `Role`. The `Role` is responsible for invoking `computeSteering` on its active `BodyMotion` component during the server tick.

```java
// Conceptual example of how a Role would drive this component
// This code would exist within the NPC's core AI tick.

// Get the active movement strategy for the current AI state
BodyMotionBase currentMotion = npc.getActiveRole().getActiveMotionController();

if (currentMotion instanceof BodyMotionTeleport) {
    // The Role provides necessary context and invokes the logic
    currentMotion.computeSteering(
        npc.getRef(),
        npc.getActiveRole(),
        npc.getSensorInfo(),
        deltaTime,
        steeringOutput,
        componentAccessor
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BodyMotionTeleport()`. The component must be configured and created via its corresponding builder, `BuilderBodyMotionTeleport`, to ensure all parameters (offsets, angles) are correctly initialized.
-   **State Manipulation:** Do not modify the internal state (e.g., `cooldown`, `tries`) from outside the class. This will break the component's internal logic and lead to unpredictable behavior.
-   **Multi-threaded Access:** As stated in the concurrency section, never call methods on this object from multiple threads. All interactions must be serialized by the main entity update loop.

## Data Pipeline
The data flow for a teleport action is a one-way command generation pipeline that terminates by adding a new component to an entity.

> Flow:
> 1.  **Input:** The `InfoProvider` supplies the position of the NPC's current target.
> 2.  **Candidate Generation:** `BodyMotionTeleport` calculates a random candidate position within a configured donut-shaped sector around the target.
> 3.  **Validation:** The candidate position is passed to the active `MotionController`, which performs collision and accessibility checks against the world geometry to find a valid, non-obstructed final destination.
> 4.  **Command Creation:** If a valid destination is found, a new `Teleport` component is instantiated, containing the destination coordinates and desired post-teleport orientation.
> 5.  **Output (Side Effect):** The `Teleport` component is added to the NPC entity using the `ComponentAccessor`.
> 6.  **Execution:** A separate ECS system, running later in the game tick, will detect the `Teleport` component and update the entity's `TransformComponent`, completing the action.

