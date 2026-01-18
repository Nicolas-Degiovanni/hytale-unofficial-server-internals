---
description: Architectural reference for MotionTimer
---

# MotionTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer
**Type:** Component / Decorator

## Definition
```java
// Signature
public abstract class MotionTimer<T extends Motion> extends MotionBase {
```

## Architecture & Concepts
The MotionTimer is a fundamental component within the server-side NPC AI framework, acting as a stateful decorator for other Motion instances. Its primary architectural purpose is to impose a temporal constraint on a wrapped behavior, effectively making any Motion time-limited.

This class enables the creation of dynamic and less predictable NPC behaviors. For example, instead of an NPC wandering indefinitely, a designer can wrap a Wander motion within a MotionTimer to create a behavior like "wander for 2 to 5 seconds". The duration is randomized within a configured range upon each activation, preventing robotic, repetitive actions.

Architecturally, MotionTimer sits between high-level AI constructs (like Behavior Trees or Finite State Machines) and the low-level steering and movement logic. It functions as a gatekeeper, allowing the decorated Motion to influence the NPC's steering only until its internal timer expires. It achieves this by delegating all lifecycle and steering computation calls to the wrapped Motion, but only if its own time-to-live has not been exceeded.

### Lifecycle & Ownership
- **Creation:** MotionTimer instances are not created directly via code. They are instantiated by the server's NPC asset loading pipeline, configured through a corresponding `BuilderMotionTimer`. This ensures that an NPC's behaviors are defined as data, not hard-coded.
- **Scope:** An instance of MotionTimer persists for the entire lifetime of the `NPCEntity` to which it belongs. However, its internal timer state is transient and is reset each time the `activate` method is called by the NPC's controlling logic.
- **Destruction:** The object is marked for garbage collection when its owning `NPCEntity` is unloaded and removed from the world. There is no manual destruction method.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains the timer's configuration (`atLeastSeconds`, `atMostSeconds`) and its active state (`activeTime`, `timeToLive`). The active state is reset upon every call to `activate`.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be accessed and modified exclusively by the single server thread responsible for ticking the corresponding NPC. Any concurrent access, particularly to `computeSteering`, will result in race conditions and undefined behavior. All interactions must be synchronized with the main server game loop.

## API Surface
The public contract of MotionTimer is primarily concerned with the standard Motion lifecycle, acting as a pass-through to the decorated component, with `computeSteering` being the notable exception.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(...) | void | O(1) | Resets the internal timer and activates the wrapped Motion. This must be called to begin a new timed execution. |
| deactivate(...) | void | O(1) | Immediately deactivates the wrapped Motion, regardless of remaining time. |
| computeSteering(...) | boolean | O(N) | The core update method, called each tick. Returns false if the timer has expired or if the wrapped Motion completes. N is the complexity of the wrapped Motion's `computeSteering`. |

## Integration Patterns

### Standard Usage
MotionTimer is not intended for direct, imperative use in game logic. It is designed to be composed within an NPC's asset definition and controlled by a higher-level AI system, such as a behavior tree.

The following conceptual example illustrates how a behavior tree might use a pre-configured MotionTimer instance.

```java
// Conceptual example within a hypothetical Behavior Tree node

// Assume 'timedWander' is a MotionTimer instance loaded with the NPC asset
Motion timedWander = npc.getMotion("timedWander");

// When this behavior becomes active, the behavior tree calls activate()
timedWander.activate(ref, role, accessor);

// On subsequent ticks, the tree calls computeSteering() until it returns false
boolean isStillRunning = timedWander.computeSteering(ref, role, info, dt, steering, accessor);

if (!isStillRunning) {
    // The timed wander is complete. Transition to the next behavior.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new MotionTimer()`. This bypasses the asset pipeline and results in an unconfigured component that will fail at runtime. All instances must be created via NPC asset builders.
- **State Manipulation:** Do not externally modify the public fields `activeTime` or `timeToLive`. Doing so will corrupt the component's internal state and break its timing logic.
- **Multi-threaded Access:** Never call methods on a MotionTimer instance from any thread other than the main server thread that owns the NPC. This will lead to severe concurrency issues.

## Data Pipeline
The MotionTimer acts as a conditional filter in the NPC's per-tick data and control flow. It intercepts the call to compute steering and only forwards it if its time has not expired.

> Flow:
> AI Controller (e.g., Behavior Tree) → `activate()` → Game Tick → `computeSteering()` → **[MotionTimer]** Time Check → `(if valid)` → Wrapped `Motion.computeSteering()` → Steering Output

