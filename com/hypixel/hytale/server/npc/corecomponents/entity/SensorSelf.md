---
description: Architectural reference for SensorSelf
---

# SensorSelf

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient

## Definition
```java
// Signature
public class SensorSelf extends SensorWithEntityFilters {
```

## Architecture & Concepts
The SensorSelf class is a specialized component within the server-side NPC AI framework. Its primary function is to enable an NPC to sense its **own** internal state, rather than external entities or environmental factors. It acts as an introspective query mechanism that answers questions like "Is my health below 20%?" or "Am I currently on fire?".

This sensor is a fundamental building block for creating reactive and state-aware behaviors. It integrates into the AI's decision-making loop (e.g., a Behavior Tree or State Machine) by evaluating a set of predefined conditions, known as filters, against the NPC entity it is attached to. If the conditions are met, the sensor is considered to have a positive "match", which can trigger a corresponding AI behavior.

Unlike other sensors that scan an area for targets, SensorSelf always targets the entity it belongs to, making it a highly efficient component for state-checking.

### Lifecycle & Ownership
- **Creation:** SensorSelf is not instantiated directly via code. It is constructed by the NPC asset loading pipeline, specifically through a corresponding `BuilderSensorSelf` instance. This process occurs when an NPC's definition, likely from a JSON or similar asset file, is parsed and its AI components are assembled.
- **Scope:** The lifecycle of a SensorSelf instance is tightly coupled to the lifecycle of the NPC entity it is a component of. It persists as long as the NPC is active in the world.
- **Destruction:** The object is eligible for garbage collection when the parent NPC entity is unloaded or destroyed. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The primary state is held within the `positionProvider` field. This object is **mutable**. The `matches` method has a critical side effect: upon a successful match, it updates the `positionProvider` with the NPC's current position from its TransformComponent. This makes the result of the sensor check available to other systems.
- **Thread Safety:** This class is **not thread-safe** and must not be considered so. All interactions with NPC components and sensors are expected to occur on the main server thread responsible for the NPC's AI tick. Unsynchronized access from other threads will lead to race conditions, inconsistent state, and server instability.

## API Surface
The public contract is minimal, designed for integration with a higher-level AI scheduler or decision-making system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(F) | Evaluates the configured filters against the NPC's own components. F is the number of filters. Returns true on a successful match and, as a side effect, updates the internal PositionProvider. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal PositionProvider, which contains the NPC's position at the time of the last successful match. |

## Integration Patterns

### Standard Usage
SensorSelf is designed to be used by a higher-level AI system that iterates over a list of sensors during an NPC's update tick. Its boolean result is typically used to control transitions in a state machine or as a condition in a behavior tree.

```java
// Within an NPC's AI update method
// This code is conceptual and illustrates the pattern

for (Sensor sensor : npc.getSensors()) {
    if (sensor.matches(npc.getRef(), npc.getRole(), dt, world.getStore())) {
        // SensorSelf matched, meaning the NPC's state meets a condition.
        // Now, retrieve the info to use in a behavior.
        InfoProvider info = sensor.getSensorInfo();

        // Trigger a behavior, e.g., a "HealSelf" action
        npc.getBehaviorController().triggerBehavior(info);
        break; // Stop checking other sensors if one has triggered
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SensorSelf()`. The component's filters and internal state are configured by the asset pipeline. Manual creation will result in a non-functional sensor.
- **State Caching:** Do not call `getSensorInfo` and cache the result for later use. The data is only valid for the exact tick in which `matches` returned true.
- **Multi-threaded Access:** Do not invoke `matches` or `getSensorInfo` from any thread other than the NPC's primary update thread. This will corrupt state and cause unpredictable behavior.

## Data Pipeline
The data flow for SensorSelf is introspective, originating from the AI tick and reading data from the NPC's own components.

> Flow:
> NPC AI Update Tick -> `SensorSelf.matches()` -> Read Self `TransformComponent` & other components -> Apply Filters -> **Update internal `PositionProvider`** -> AI System reads `PositionProvider` -> Behavior Execution

