---
description: Architectural reference for SensorFlag
---

# SensorFlag

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient

## Definition
```java
// Signature
public class SensorFlag extends SensorBase {
```

## Architecture & Concepts
The SensorFlag is a fundamental component within the server-side NPC AI perception system. It functions as a simple, data-driven predicate that evaluates a specific boolean flag within an NPC's state.

Architecturally, it is one of the most common types of *Sensors*. A Sensor is a stateless condition that the AI's "brain" or behavior tree evaluates on each tick to determine if a state transition should occur. The SensorFlag allows game designers to create complex AI behaviors that react to custom, game-specific states (e.g., "isAlerted", "hasCompletedObjective", "isWet") without requiring new engine code.

It acts as a bridge between an NPC's internal state, represented by the **Role** component, and the decision-making logic of its behavior controller. Its primary design goal is performance and reusability; a single SensorFlag instance defined in an NPC asset is shared across all instances of that NPC type.

## Lifecycle & Ownership
- **Creation:** SensorFlag instances are not created directly. They are instantiated by the NPC asset loading pipeline via a corresponding **BuilderSensorFlag**. This builder is configured from a data source, typically a JSON asset file defining the NPC's complete component and AI structure.
- **Scope:** An instance of SensorFlag persists for the lifetime of the NPC asset definition in memory. It is considered a static, shared component of an NPC archetype.
- **Destruction:** The object is eligible for garbage collection only when its parent NPC asset definition is unloaded from the server, for instance during a zone unload or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** Immutable. The internal fields **flagIndex** and **value** are final and are set exclusively at construction time. A SensorFlag instance never changes its state after creation. It holds no per-NPC instance data.
- **Thread Safety:** This class is inherently thread-safe. The `matches` method is a pure function with respect to the object's internal state.
    - **WARNING:** While the SensorFlag object itself is safe, the components it operates on (**Ref<EntityStore>**, **Role**) are typically bound to a specific world thread. The calling system, such as an AI scheduler or behavior tree executor, is responsible for ensuring that calls to `matches` are made from the correct thread and that no data races occur on the Role component's flag set.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the sensor's condition. Returns true if the flag at `flagIndex` on the provided Role component is equal to the configured `value`. |
| getSensorInfo() | InfoProvider | O(1) | Returns null. This sensor does not provide extended debug information to the AI visualization system. |

## Integration Patterns

### Standard Usage
A SensorFlag is never used in isolation. It is attached to an NPC's definition and evaluated by the core AI update loop or a behavior tree. The system retrieves the sensor from the NPC's definition and invokes `matches` to drive logic.

```java
// Hypothetical usage within an AI Behavior Tree node

// The 'sensor' instance is pre-loaded with the NPC asset
SensorFlag sensor = npc.getBehavior().getSensor("IsAlertedSensor");

// During the AI tick, the node evaluates the sensor
boolean isAlerted = sensor.matches(npc.getEntityRef(), npc.getRole(), deltaTime, world.getEntityStore());

if (isAlerted) {
    // Transition to the "Combat" behavior branch
    behaviorTree.transitionTo(State.COMBAT);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to construct a SensorFlag manually. The `flagIndex` is resolved from asset data via the **BuilderSupport** context. Manual creation would lead to an invalid or incorrect index, causing unpredictable AI behavior. Always define sensors within the NPC's asset files.
- **Stateful Subclassing:** Do not extend SensorFlag to add mutable state. The sensor system is designed around stateless, reusable predicates. Introducing state would break thread-safety assumptions and violate the component's architectural contract.

## Data Pipeline
The SensorFlag is not part of a data processing pipeline but rather a control flow pipeline. It translates a static configuration into a runtime decision.

> Flow:
> NPC Asset File (JSON) -> Asset Loader -> **BuilderSensorFlag** -> **SensorFlag Instance** (in memory) -> AI Update Tick -> `matches(Role)` -> Boolean Result -> Behavior Tree State Change

