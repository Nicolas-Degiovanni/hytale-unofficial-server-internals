---
description: Architectural reference for MotionBase
---

# MotionBase

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Component Base Class

## Definition
```java
// Signature
public abstract class MotionBase extends AnnotatedComponentBase implements Motion {
```

## Architecture & Concepts
The **MotionBase** class is an abstract foundational component within the server-side Non-Player Character (NPC) AI framework. It serves as the common ancestor for all concrete motion instructions, establishing a standardized contract for behaviors that involve the physical movement of an NPC entity.

Architecturally, this class embodies the *Template Method* design pattern. It provides shared functionality inherited from **AnnotatedComponentBase** while delegating the specific implementation of movement logic (e.g., walking, flying, swimming) to its concrete subclasses. By implementing the **Motion** interface, it guarantees that all motion components can be handled polymorphically by the NPC's core behavior system, such as a state machine or behavior tree.

This component is a critical link between high-level AI decisions (e.g., "patrol the area") and low-level entity physics (e.g., updating velocity and position).

### Lifecycle & Ownership
- **Creation:** **MotionBase** itself is never instantiated directly due to its abstract nature. Concrete subclasses (e.g., **WalkToTargetMotion**, **FollowEntityMotion**) are instantiated by higher-level AI controllers or instruction factories when an NPC is commanded to perform a specific movement task.
- **Scope:** The lifetime of a **MotionBase** instance is ephemeral and tightly coupled to the duration of the specific AI instruction it represents. It exists only as long as the NPC is actively executing that particular motion.
- **Destruction:** The object is eligible for garbage collection as soon as the motion is completed, cancelled, or superseded by a new instruction. Ownership is managed by the NPC's behavior controller, which discards its reference upon task completion.

## Internal State & Concurrency
- **State:** While **MotionBase** itself may contain minimal state, its concrete subclasses are inherently stateful. They must track mutable data such as target coordinates, pathing information, and progress towards the goal. This state is modified on each server tick as the NPC executes the motion.
- **Thread Safety:** This component is **not thread-safe**. All interactions with a **MotionBase** instance and its subclasses must be performed exclusively on the main server thread that processes game ticks. Modifying or accessing its state from an asynchronous task or a different thread will lead to severe concurrency issues, including data corruption and server instability.

## API Surface
The public API is primarily defined by the **Motion** interface and inherited from **AnnotatedComponentBase**. The following represents a logical interpretation of its contract, as concrete methods reside in subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(npc, deltaTime) | void | O(N) | Called once per server tick to update the NPC's state. Complexity depends on the pathfinding algorithm used. |
| isFinished() | boolean | O(1) | Returns true if the motion has been successfully completed. |
| onStart(npc) | void | O(1) | Initialization logic executed once when the motion becomes active. |
| onEnd(npc) | void | O(1) | Cleanup logic executed once when the motion is completed or interrupted. |

## Integration Patterns

### Standard Usage
Developers should not interact with **MotionBase** directly. The standard pattern is to extend it to create a new, specific type of movement behavior. This new class is then instantiated and managed by the NPC's AI system.

```java
// 1. Define a concrete motion by extending MotionBase
public class WalkToLocation extends MotionBase {
    // Implementation of pathfinding and movement logic...
    @Override
    public void tick(NPC npc, float deltaTime) {
        // Update NPC position towards a target
    }
    // ...
}

// 2. An AI system assigns the motion to an NPC
WalkToLocation instruction = new WalkToLocation(targetPosition);
npc.getBehaviorController().setInstruction(instruction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** It is a compile-time error to attempt `new MotionBase()`. This class exists only to be extended.
- **State Manipulation from External Systems:** Never modify the internal state of an active motion component from outside the component's own *tick* method. For example, do not change its target destination directly; instead, cancel the current motion and issue a new one. This ensures the component's internal state machine remains consistent.
- **Asynchronous Execution:** Do not attempt to execute motion logic off the main server thread. All pathfinding calculations and position updates must be synchronized with the server's game loop.

## Data Pipeline
The flow of control and data is managed by the server's core NPC update loop.

> Flow:
> NPC Behavior Tree/FSM -> **Instantiates Concrete Motion** -> NPC Controller executes `onStart()` -> Server Game Loop calls `tick()` on active Motion -> **Motion updates NPC Entity's Position/Velocity** -> Physics Engine resolves movement -> State is replicated to clients

