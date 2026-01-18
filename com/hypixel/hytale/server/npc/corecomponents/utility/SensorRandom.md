---
description: Architectural reference for SensorRandom
---

# SensorRandom

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorRandom extends SensorBase {
```

## Architecture & Concepts
The SensorRandom class is a fundamental component within the server-side NPC Artificial Intelligence framework. Its primary function is to act as a non-deterministic signal generator, injecting variability and natural-feeling randomness into NPC behavior.

Architecturally, it serves as a leaf node within a larger decision-making structure, such as a Behavior Tree or a Finite State Machine. It operates as a stateful, time-based switch that oscillates between a *true* and *false* state. The duration of each state is randomized within configurable bounds, allowing designers to control the frequency and length of the behaviors it governs. For example, it could be used to decide when an NPC should start or stop a random idle animation, look around, or wander to a new position.

This component decouples the concept of random timing from the core behavior logic, promoting cleaner and more reusable AI design.

## Lifecycle & Ownership
- **Creation:** A SensorRandom instance is never created directly using its constructor. It is instantiated by the server's NPC asset loading pipeline. An associated BuilderSensorRandom, typically defined in a data file like JSON, is processed by a factory which then constructs the SensorRandom component and attaches it to an NPC entity.
- **Scope:** The lifecycle of a SensorRandom object is strictly bound to the lifecycle of its parent NPC entity. It is created when the NPC is spawned into the world and persists for the NPC's entire existence.
- **Destruction:** The object is eligible for garbage collection when the parent NPC entity is despawned or destroyed. It holds no external resources and does not require an explicit destruction method.

## Internal State & Concurrency
- **State:** This component is highly **mutable**. Its internal fields, *remainingDuration* and *state*, are modified on nearly every invocation of the *matches* method to track the countdown and the current output. The configuration parameters (min/max durations) are immutable after construction.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated exclusively by the game loop thread responsible for its parent NPC. Any concurrent access to the *matches* method from other threads will introduce race conditions on its internal state, leading to corrupted timers and unpredictable AI behavior. All interactions must be synchronized with the owning NPC's update tick.

## API Surface
The public contract is minimal, centered on the periodic evaluation of the sensor's state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the sensor for the current game tick. This method decrements an internal timer by delta time (dt). If the timer expires, the sensor's boolean state is flipped and a new random duration is chosen. Returns the current state. |
| getSensorInfo() | InfoProvider | O(1) | Returns debugging information for visualization tools. The current implementation returns null. |

## Integration Patterns

### Standard Usage
A developer or designer will not interact with this class directly in code. Instead, it is configured declaratively within an NPC's asset definition file. The game engine's AI system then invokes the *matches* method automatically during the NPC's update cycle.

```java
// This code is conceptual and resides deep within the AI engine.
// It is not intended for typical user implementation.

// During an NPC's AI update tick...
boolean isSignalActive = this.sensorRandomComponent.matches(entityRef, role, deltaTime, worldStore);

// A Behavior Tree node might then use this result
if (isSignalActive) {
    // Execute the "Wander" behavior branch
} else {
    // Execute the "StandStill" behavior branch
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorRandom(...)`. The component must be instantiated via the asset pipeline to ensure its configuration is loaded correctly. Failure to do so will result in a misconfigured or non-functional sensor.
- **State Tampering:** Do not use reflection or other means to externally modify the internal *state* or *remainingDuration* fields. This will break the component's internal logic and lead to unpredictable timing.
- **Ignoring Delta Time:** Calling the *matches* method with a static or incorrect delta time value will cause the timer to progress incorrectly, either too fast or not at all.

## Data Pipeline
The flow for this component is one of configuration and control, not traditional data processing.

> Flow:
> NPC Asset File (JSON) -> Asset Loader -> **BuilderSensorRandom** -> NPC Factory -> **SensorRandom Instance** -> AI Behavior Tree -> `matches()` call -> Boolean Result -> AI Decision

