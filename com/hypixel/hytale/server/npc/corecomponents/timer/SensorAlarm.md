---
description: Architectural reference for SensorAlarm
---

# SensorAlarm

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorAlarm extends SensorBase {
```

## Architecture & Concepts
The SensorAlarm is a fundamental component within the Hytale NPC Behavior System, acting as a conditional node or a "gate" in an NPC's behavior tree. Its sole purpose is to evaluate the state of a shared, named Alarm object against the current world time, enabling time-based NPC decision-making.

This class serves as the bridge between the abstract logic of a behavior tree (e.g., "Has enough time passed?") and the concrete state of the game world, specifically the WorldTimeResource. It is not a general-purpose timer but a highly specific state checker.

Architecturally, SensorAlarm instances are declarative. They are defined in NPC asset files and instantiated by the asset loading system via a corresponding BuilderSensorAlarm. This design decouples the NPC's behavior logic from its implementation, allowing designers to create complex, time-sensitive AI without writing code.

## Lifecycle & Ownership
- **Creation:** A SensorAlarm is instantiated by the BuilderSensorAlarm when an NPC's behavior asset is loaded and parsed by the server. This typically occurs when an NPC entity is spawned into the world and its behavior tree is constructed. The builder resolves dependencies, such as linking this sensor to a specific Alarm instance defined elsewhere in the NPC's configuration.
- **Scope:** The object's lifetime is tightly coupled to the NPC's behavior tree instance. It persists as long as the NPC entity exists and is governed by that specific behavior configuration.
- **Destruction:** The object is marked for garbage collection when the parent NPC entity is despawned or its behavior tree is replaced. It has no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** SensorAlarm is a stateful component. Its configuration—the target Alarm, the State to check for, and the clear flag—is immutable after creation. However, it operates on and can **mutate** the external Alarm object it references. This side effect is a critical aspect of its design, particularly when the clear flag is true.
- **Thread Safety:** **This component is not thread-safe and must not be accessed concurrently.** All interactions are expected to occur on the primary server thread that processes NPC updates. The `matches` method performs a read-modify-write operation on the shared Alarm when the `clear` flag is set, which would create a severe race condition if invoked from multiple threads.

## API Surface
The public contract is minimal, focused entirely on its evaluation within the behavior system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the configured Alarm state against the current game time. **Warning:** This method can have side effects. If the sensor is configured to `clear` on a `PASSED` state, it will mutate the underlying Alarm object, resetting it. |

## Integration Patterns

### Standard Usage
This component is not designed for direct invocation by developers. It is exclusively managed and executed by the NPC's behavior tree engine during the entity update tick. A game designer would configure it declaratively in an asset file.

The conceptual engine-level usage is as follows:

```java
// Engine-level conceptual usage.
// Do not invoke this component manually.

// Inside the NPC's behavior tree update loop:
boolean conditionMet = someSensorAlarm.matches(entityRef, currentRole, deltaTime, worldStore);

if (conditionMet) {
    // The engine proceeds down this branch of the behavior tree.
} else {
    // The engine explores a different branch.
}
```

### Anti-Patterns (Do NOT do this)
- **Bypassing the Builder:** Do not attempt to construct a SensorAlarm manually. It relies on the builder and the BuilderSupport context to correctly resolve its Alarm dependency from the NPC's component set.
- **State Mutation:** Do not use reflection or other means to modify the internal `alarm`, `state`, or `clear` fields after construction. An instance's configuration is intended to be immutable.
- **Asynchronous Evaluation:** Never call the `matches` method from a separate thread or asynchronous task. All NPC logic, including sensor evaluation, must be synchronized with the main server tick to prevent state corruption.

## Data Pipeline
The data flow for this component is a simple transformation from global game state into a binary decision for the AI system.

> Flow:
> WorldTimeResource -> **SensorAlarm.matches()** -> `boolean` Result -> Behavior Tree Executor -> NPC Action Selection

