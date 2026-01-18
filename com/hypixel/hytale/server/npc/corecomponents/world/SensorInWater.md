---
description: Architectural reference for SensorInWater
---

# SensorInWater

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class SensorInWater extends SensorBase {
```

## Architecture & Concepts
The SensorInWater is a highly specialized component within the server-side NPC Artificial Intelligence framework. It functions as a simple, boolean-state predicate, designed to answer a single, critical question: "Is the NPC entity currently submerged in water?".

As a subclass of SensorBase, it inherits foundational sensor behaviors such as cooldowns and activation toggles. However, its primary role is to provide a concrete condition check that higher-level AI constructs, such as Behavior Trees or Goal-Oriented Action Planners, can evaluate. These systems query the SensorInWater to determine if water-specific behaviors, like swimming or surfacing for air, should be activated.

This class acts as a direct bridge between the NPC's physical state, managed by its MotionController, and its logical decision-making process. It encapsulates the specific check, decoupling the AI logic from the underlying physics and world-interaction implementation.

### Lifecycle & Ownership
- **Creation:** SensorInWater instances are not created manually. They are instantiated by a data-driven configuration system when an NPC's AI profile is loaded. The `BuilderSensorInWater` is used during this deserialization process to construct the sensor as part of a larger AI behavior graph.
- **Scope:** The lifecycle of a SensorInWater instance is tightly bound to the lifecycle of the NPC entity it belongs to. It persists as a component within the NPC's `Role` for the entire duration of the NPC's existence in the world.
- **Destruction:** The object is marked for garbage collection when the parent NPC entity is unloaded or destroyed. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This class is fundamentally **stateless**. It stores no data itself. Its evaluation is a pure function of the state of the `Role` object passed into the `matches` method on each tick. Any stateful behavior, such as evaluation cooldowns, is managed by the parent SensorBase class.
- **Thread Safety:** This component is **not thread-safe** and must not be considered as such. It is designed to be invoked exclusively from the main server thread during the synchronous NPC update tick. Accessing the `matches` method from a concurrent thread will lead to race conditions with the underlying entity's MotionController and result in undefined behavior.

## API Surface
The public contract is minimal, consisting only of the methods required to fulfill its role as a Sensor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates if the NPC is in water. Returns true if the base sensor conditions are met and the NPC's active MotionController reports an `inWater` state. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly. Instead, it is defined within an NPC's asset files (e.g., JSON) and evaluated automatically by the AI's root behavior node. The system internally polls its list of sensors to make decisions.

A conceptual example of how the system uses the sensor:
```java
// Hypothetical AI logic within a Behavior Tree node
// This code does NOT exist; it is for conceptual understanding only.

boolean canSwim = aIController.evaluateSensor(SensorInWater.class);

if (canSwim) {
    // Transition to the "SwimmingBehavior" state
    aIController.setActiveBehavior(swimmingBehavior);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorInWater()`. The AI system relies on the builder pattern and data-driven configuration to construct and wire sensors correctly. Manual instantiation will result in a component that is not registered with the AI controller.
- **External Invocation:** Do not call the `matches` method from outside the NPC's core AI update loop. The state of the `Role` and `MotionController` is only guaranteed to be valid and consistent during that specific phase of the server tick.
- **Stateful Subclassing:** Do not extend this class to add state. Sensors are intended to be stateless predicates. If you need to store information related to being in water, use a separate Memory or State component.

## Data Pipeline
The flow of data for a single evaluation is linear and synchronous, occurring entirely within one server tick.

> Flow:
> Server Tick -> NPC Update Phase -> AI Behavior Evaluation -> **SensorInWater.matches()** -> Query `role.getActiveMotionController()` -> Return `inWater()` boolean -> AI makes decision

