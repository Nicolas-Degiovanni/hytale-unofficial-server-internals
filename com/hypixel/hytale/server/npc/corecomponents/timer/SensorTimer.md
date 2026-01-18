---
description: Architectural reference for SensorTimer
---

# SensorTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer
**Type:** Transient

## Definition
```java
// Signature
public class SensorTimer extends SensorBase {
```

## Architecture & Concepts
The SensorTimer is a conditional component within the server-side NPC AI framework. It functions as a specific *predicate* that evaluates the state of a named Timer component associated with an NPC. Its primary role is to enable time-based decision-making in NPC behavior trees and state machines.

Architecturally, SensorTimer acts as a bridge between a generic, low-level Timer component and the higher-level behavioral logic defined in an NPC's Role. It is a data-driven component, meaning its parameters—such as the target timer, the required state (e.g., RUNNING, STOPPED), and the time-remaining thresholds—are configured in NPC asset files, not hardcoded. This allows designers to create complex, time-sensitive behaviors without modifying engine code.

For example, a SensorTimer can be used to ask questions like:
- "Has the 'abilityCooldown' timer finished?"
- "Is the 'chargeUp' timer currently running and has been for 1 to 2.5 seconds?"
- "Is the 'stun' timer currently stopped?"

The system evaluates this sensor during the NPC's update tick. The boolean result of its `matches` method determines whether a specific behavioral branch is taken.

### Lifecycle & Ownership
- **Creation:** SensorTimer is instantiated exclusively by the NPC asset loading pipeline. A corresponding `BuilderSensorTimer` reads configuration data from an NPC's definition file and, using a `BuilderSupport` context, constructs the SensorTimer instance. It is never created manually.
- **Scope:** The lifecycle of a SensorTimer instance is tightly coupled to the lifecycle of the NPC's `Role` or behavior tree that contains it. It persists as long as the NPC's behavior configuration is active. Each NPC has its own distinct instances of its sensors.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the parent NPC entity is unloaded or its behavior definition is replaced.

## Internal State & Concurrency
- **State:** The configuration of a SensorTimer instance (minTimeRemaining, maxTimeRemaining, timerState) is immutable after creation. However, its core function is to read the external, mutable state of a `Timer` object. Therefore, its output is stateful, dependent on the current value and state of the referenced timer at the moment of evaluation.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be operated exclusively by the single thread responsible for updating a specific NPC's AI tick. The underlying `Timer` it references is expected to be managed by the same thread. Accessing a SensorTimer or its associated `Timer` from a concurrent thread will result in undefined behavior and race conditions.

## API Surface
The public contract is focused entirely on evaluation within the AI system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the associated Timer against the configured state and time range. Returns true if conditions are met. |

## Integration Patterns

### Standard Usage
Direct interaction with SensorTimer is rare. Its primary integration is declarative, through an NPC's asset definition file. The game engine's AI system is responsible for invoking the `matches` method at the appropriate time.

Below is a conceptual example of how a SensorTimer would be defined in a JSON-like asset file.

```json
// Conceptual NPC Asset Definition
{
  "id": "hostile_golem",
  "behaviors": [
    {
      "name": "ChargeAttack",
      "conditions": [
        {
          "type": "SensorTimer",
          "timer": "charge_up_timer",
          "state": "RUNNING",
          "remainingTime": [1.0, 1.5] // Fire attack when timer is between 1.0 and 1.5s
        }
      ],
      "actions": [ ... ]
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SensorTimer()`. The component requires a fully configured builder and a `BuilderSupport` context to correctly resolve its `Timer` dependency from the NPC's component store. Bypassing this process will result in a non-functional sensor.
- **External State Manipulation:** Do not attempt to get a reference to a SensorTimer and call its methods from outside the NPC's standard AI update loop. The evaluation is only valid within the context of a single, atomic game tick.

## Data Pipeline
The flow of data begins with the asset definition and ends with a boolean decision during the game loop.

> Flow:
> NPC Asset File -> `BuilderSensorTimer` -> **SensorTimer Instance** -> NPC Role Update -> `SensorTimer.matches()` -> Boolean Decision -> Behavior Tree Execution

