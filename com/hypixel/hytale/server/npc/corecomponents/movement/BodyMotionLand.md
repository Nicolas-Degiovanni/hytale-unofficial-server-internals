---
description: Architectural reference for BodyMotionLand
---

# BodyMotionLand

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Component/Strategy

## Definition
```java
// Signature
public class BodyMotionLand extends BodyMotionFind {
```

## Architecture & Concepts
The BodyMotionLand class is a specialized movement strategy component within the server-side NPC AI framework. Its primary architectural role is to act as a **terminal condition handler** for aerial movement. While a more generic component like BodyMotionFind calculates a path towards a target, BodyMotionLand determines the precise moment an NPC should transition from a flying state to a walking or landed state.

It extends BodyMotionFind, inheriting the fundamental logic for target-seeking and path evaluation. BodyMotionLand overrides key methods to inject landing-specific criteria, such as vertical proximity and horizontal leniency. This component is crucial for creating believable NPC behaviors, preventing flying entities from overshooting their targets or getting stuck in a perpetual flight state when their destination is on the ground.

It operates within the context of an NPC's current Role and directly manipulates the active MotionController, effectively serving as a state transition trigger within the NPC's movement state machine.

### Lifecycle & Ownership
- **Creation:** BodyMotionLand instances are not created directly via code. They are instantiated by the NPC asset pipeline through a corresponding builder, BuilderBodyMotionLand, during server startup or when NPC definitions are loaded. Its parameters, such as goal leniency, are defined declaratively in NPC configuration files.
- **Scope:** An instance of BodyMotionLand is scoped to the NPC behavior definition it is part of. It persists for the lifetime of the server session, as it is part of the static, configured definition of an NPC's capabilities.
- **Destruction:** The object is eligible for garbage collection only when its parent NPC definition is unloaded from the server.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable** after construction. Its internal fields, such as goalLenience, are marked as final and are initialized once by its builder. It does not cache or modify its own state during method execution. All calculations are performed using state passed in via method arguments, primarily from the entity's components.
- **Thread Safety:** This class is not inherently thread-safe. It is designed to be executed by the single-threaded AI update loop for a specific entity. Invoking its methods concurrently on the same entity from multiple threads will lead to race conditions and undefined behavior, particularly when modifying the entity's MotionController. All interactions must be synchronized by the calling game loop.

## API Surface
The public API is designed for internal consumption by the NPC Role and behavior systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSteering(...) | boolean | O(1) | Calculates steering towards a target. Critically, if landing conditions are met, it triggers a state change to the "Walk" motion controller and returns false to terminate further steering calculations for the current behavior. |
| canComputeMotion(...) | boolean | O(1) | A predicate used by the AI system to determine if this component should be active. Returns true only if the NPC is currently using a MotionControllerFly, ensuring it only runs during flight. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in gameplay code. Its behavior is defined in NPC asset files and invoked automatically by the AI's Role system. The system internally performs a sequence similar to the one below during an AI tick.

```java
// Conceptual example of internal engine usage
// This code is managed by the Role system, not by developers.

Role activeRole = npc.getActiveRole();
BodyMotionLand motionLogic = activeRole.getMovementComponent(BodyMotionLand.class);

// The engine first checks if this logic is applicable
if (motionLogic.canComputeMotion(entityRef, activeRole, ...)) {
    
    // If applicable, it computes the next steering instruction
    Steering desiredSteering = new Steering();
    boolean shouldContinue = motionLogic.computeSteering(entityRef, activeRole, ..., desiredSteering);

    if (!shouldContinue) {
        // The component has triggered a state change (e.g., landing)
        // and the current behavior is now complete.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BodyMotionLand()`. The component requires configuration parameters that are injected by its corresponding builder during the asset loading process. Direct instantiation will result in a misconfigured and non-functional component.
- **Ignoring the Guard Condition:** Do not call `computeSteering` without a preceding successful call to `canComputeMotion`. The `computeSteering` method assumes the NPC is in a flying state; calling it at other times can lead to assertion errors or invalid state transitions.
- **Stateful Decoration:** Do not wrap this class in another class that adds state. Its stateless nature is critical for predictable behavior within the AI system.

## Data Pipeline
BodyMotionLand acts as a conditional processor and state transition node in the NPC movement pipeline. It consumes entity state and, upon meeting its conditions, alters the entity's core motion controller.

> Flow:
> AI Tick -> Role System Evaluates Behaviors -> **BodyMotionLand.canComputeMotion()** returns true -> **BodyMotionLand.computeSteering()** -> Internal call to **isGoalReached()** -> Condition Met (target is close and below) -> Role.setActiveMotionController("Walk") -> Entity State Change -> Locomotion System uses new walking controller on next tick.

