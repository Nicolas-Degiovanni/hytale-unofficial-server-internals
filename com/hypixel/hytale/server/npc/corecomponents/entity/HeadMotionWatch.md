---
description: Architectural reference for HeadMotionWatch
---

# HeadMotionWatch

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Component / Strategy Object

## Definition
```java
// Signature
public class HeadMotionWatch extends HeadMotionBase {
```

## Architecture & Concepts
The HeadMotionWatch class is a server-side NPC behavior component that implements a specific head-turning strategy: orienting an NPC's head to look directly at a target. It functions as a pure data processor within the broader NPC AI and movement system.

Its core responsibility is to translate sensory information about a target's position into a desired rotational state, represented by a Steering object. This class does not enact movement itself; rather, it calculates the goal orientation (yaw and pitch). A separate, higher-level system, such as the NPC physics or animation controller, consumes the populated Steering object to smoothly apply the rotation to the NPC's model over subsequent game ticks.

This component is designed to be a modular and reusable piece of logic. It decouples the act of sensing a target from the calculation of how to look at it, allowing different sensor types to feed into this standardized "watch" behavior.

### Lifecycle & Ownership
- **Creation:** An instance of HeadMotionWatch is not created directly. It is instantiated by the NPC asset loading pipeline via its corresponding builder, BuilderHeadMotionWatch, when an NPC is spawned. This process reads from the NPC's definition files to configure its behaviors.
- **Scope:** The object's lifetime is tightly coupled to the NPC entity that owns it. It persists as a component of the NPC for as long as the NPC exists in the world.
- **Destruction:** The object is eligible for garbage collection when its owning NPC entity is despawned or destroyed. It has no explicit cleanup or destruction method.

## Internal State & Concurrency
- **State:** This class is effectively immutable after construction. Its only internal state, relativeTurnSpeed, is a final double initialized in the constructor. The primary method, computeSteering, is stateless and its output is a pure function of its arguments for any given game tick.
- **Thread Safety:** The class itself is thread-safe due to its immutable state. However, its usage is **not** thread-safe. It is designed to be invoked exclusively from the main server thread that governs the game world and entity updates. The ComponentAccessor and EntityStore references it operates on are not safe for concurrent access.

    **WARNING:** Calling computeSteering from any thread other than the main server thread will lead to severe data corruption, race conditions, and unpredictable server instability.

## API Surface
The public contract is focused entirely on the steering computation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSteering(...) | boolean | O(1) | Calculates the target yaw and pitch to face a target. Modifies the `desiredSteering` output parameter with the result. Returns true if a target was available and processed. |

## Integration Patterns

### Standard Usage
This component is not intended for direct invocation by developers. It is automatically managed and invoked by the NPC's core AI loop, typically as part of a Role or behavior tree. The system provides the necessary context on each tick.

A simplified view of how the system might use it:
```java
// Executed by the NPC's core update loop
void processNpcHeadMotion(Npc npc, double dt) {
    // ... system retrieves current sensor data and steering object ...
    InfoProvider sensorInfo = npc.getActiveSensorInfo();
    Steering currentSteering = npc.getSteeringOutput();
    HeadMotionBase headMotion = npc.getActiveHeadMotion(); // This could be a HeadMotionWatch instance

    if (headMotion != null) {
        headMotion.computeSteering(
            npc.getRef(),
            npc.getCurrentRole(),
            sensorInfo,
            dt,
            currentSteering,
            world.getComponentAccessor()
        );
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new HeadMotionWatch()`. This bypasses the configuration pipeline that sets critical parameters like `relativeTurnSpeed` from the NPC's asset files. Always define NPC behaviors declaratively in assets.
- **State Caching:** Do not extend this class to cache results or state between calls. The component is designed to be stateless and re-entrant. Caching can lead to stale targeting information.
- **Misuse for Translation:** This component explicitly clears translational steering. Do not attempt to use it to make an NPC move towards a target; its sole purpose is rotational head tracking.

## Data Pipeline
HeadMotionWatch acts as a transformation stage in the NPC behavior data flow. It converts high-level sensory data into low-level directional commands.

> Flow:
> NPC Sensor System → InfoProvider (Target Position) → **HeadMotionWatch** → Steering (Desired Yaw/Pitch) → NPC Physics & Animation System → TransformComponent Update

