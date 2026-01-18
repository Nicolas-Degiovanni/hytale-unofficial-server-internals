---
description: Architectural reference for BodyMotionTimer
---

# BodyMotionTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionTimer extends MotionTimer<BodyMotion> implements BodyMotion {
```

## Architecture & Concepts
The BodyMotionTimer is a specialized, stateful component that decorates a core BodyMotion object with time-based execution constraints. It is a fundamental building block within the server-side NPC instruction and behavior system.

Architecturally, this class implements the **Decorator Pattern**. By extending MotionTimer and implementing the BodyMotion interface, it can be used interchangeably with a standard BodyMotion object. This allows the NPC's AI controller to treat timed and non-timed motions uniformly, while the underlying timer transparently manages the motion's lifecycle.

Its primary role is to gate the execution of an NPC body animation or posture, ensuring it persists for a configured duration. It acts as a state machine for a single, timed action, transitioning from *running* to *completed* after a set interval. This component is crucial for creating complex, sequential behaviors in NPC AI, such as pausing, performing a specific animation for N seconds, or holding a pose.

## Lifecycle & Ownership
- **Creation:** BodyMotionTimer instances are not intended for direct instantiation by game logic. They are constructed exclusively by the NPC asset pipeline, specifically via a BuilderBodyMotionTimer. This builder is invoked during server startup or when NPC definitions are loaded, using a BuilderSupport context to resolve dependencies.
- **Scope:** The lifetime of a BodyMotionTimer is ephemeral and strictly bound to the execution of the single motion it manages. An instance is created for one-time use by the NPC's behavior controller.
- **Destruction:** The object is eligible for garbage collection as soon as the NPC's AI transitions to a new state or instruction, dropping all references to the completed timer. These are high-frequency, short-lived objects.

## Internal State & Concurrency
- **State:** This class is **highly mutable**. It inherits state from MotionTimer, which tracks the elapsed time, duration, and completion status of the motion. It also holds an immutable reference to the decorated BodyMotion object.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. All interactions, including creation, updates (ticking), and state checks, are expected to occur on the main server thread responsible for the corresponding NPC's AI updates. Unsynchronized access will lead to race conditions and undefined behavior.

## API Surface
The primary API contract is inherited from the MotionTimer base class and the BodyMotion interface. The method defined directly in this class is for interface compliance.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSteeringMotion() | BodyMotion | O(1) | Fulfills the BodyMotion interface contract by delegating the call to the wrapped motion object. |

**Warning:** The most critical methods, such as `update(deltaTime)` and `isFinished()`, are inherited from MotionTimer and constitute the core operational contract for this class.

## Integration Patterns

### Standard Usage
The BodyMotionTimer is not used directly. Instead, it is executed by a higher-level system, such as an NPC's behavior tree or instruction queue. The controlling system retrieves the pre-configured timer and ticks it each server update cycle.

```java
// Pseudo-code for an NPC AI Controller
// The 'currentMotion' is a BodyMotionTimer instance, pre-built from assets.
if (currentMotion instanceof MotionTimer) {
    MotionTimer timer = (MotionTimer) currentMotion;
    
    // Update the timer with the frame's delta time
    timer.update(deltaTime);

    // If the timed motion is complete, advance to the next behavior
    if (timer.isFinished()) {
        this.advanceToNextInstruction();
    }
}

// Apply the motion to the NPC entity for the current frame
npc.applyBodyMotion(currentMotion);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BodyMotionTimer(...)`. The object is complex to configure and requires a BuilderSupport context provided only by the asset loading system. Manual creation will result in a misconfigured, non-functional component.
- **State Resetting:** These timers are single-use objects. Do not attempt to reset their internal state for reuse. Once a timer is finished, it must be discarded and a new one retrieved for the next execution of the motion.
- **External State Modification:** Do not attempt to modify the internal timer state inherited from MotionTimer. Doing so will corrupt the component's behavior and break the NPC's instruction sequencing.

## Data Pipeline
The BodyMotionTimer functions as a control-flow component within the NPC behavior pipeline. It does not transform data, but rather gates the flow of control based on time.

> Flow:
> NPC Behavior Tree Node -> Selects Timed Motion -> **BodyMotionTimer.update()** -> Check `isFinished()` -> If false, apply `BodyMotion` to NPC Entity / If true, transition to next Behavior Tree Node

