---
description: Architectural reference for SensorMotionController
---

# SensorMotionController

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient

## Definition
```java
// Signature
public class SensorMotionController extends SensorBase {
```

## Architecture & Concepts
The SensorMotionController is a specialized conditional component within the server-side NPC artificial intelligence framework. It functions as a specific type of **Sensor**, a class whose primary purpose is to evaluate a condition about the game world or the NPC's internal state and return a boolean result.

This sensor's sole responsibility is to answer the question: **"Is the NPC currently using a specific motion controller?"** For example, it can be configured to check if an NPC's active motion controller is "swimming", "flying", or "climbing".

Architecturally, it serves as a leaf node or a conditional guard in a larger behavior graph, such as a Behavior Tree or a Finite State Machine. By composing these sensors, designers can create complex, state-aware behaviors that prevent NPCs from attempting actions in inappropriate contexts (e.g., preventing a "charge" attack while the NPC is in a "swimming" state). It inherits foundational logic from SensorBase, which likely handles more generic conditions like entity validity or range checks.

### Lifecycle & Ownership
- **Creation:** An instance is created exclusively via its corresponding builder, BuilderSensorMotionController. This typically occurs during the server's boot process when NPC behavior profiles are loaded and parsed from configuration files. It is not intended for on-the-fly instantiation during the game loop.
- **Scope:** The object's lifetime is tied to the NPC behavior definition it is part of. It is effectively a static, configured piece of a larger AI logic graph and persists as long as that definition is loaded in memory.
- **Destruction:** The object is managed by the Java garbage collector. It is reclaimed when the parent NPC behavior definition is unloaded, for instance, during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** The component's state is immutable after construction. The core state, the `motionControllerName` string, is set once in the constructor and cannot be changed. This design ensures predictable and repeatable behavior evaluation.
- **Thread Safety:** The object itself is inherently thread-safe due to its immutable state. However, the `matches` method operates on mutable game state objects passed as parameters (e.g., Role, EntityStore). The calling context, typically the main server thread for a given world region, is responsible for ensuring that calls to `matches` are not concurrent for the same NPC entity. It is a design violation to invoke this method from asynchronous tasks or worker threads without external synchronization.

## API Surface
The public contract is minimal, exposing only the evaluation logic inherited and specialized from SensorBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the sensor's condition. Returns true only if the base sensor conditions are met AND the NPC's active motion controller type matches the name configured for this sensor (case-insensitive). |
| getSensorInfo() | InfoProvider | O(1) | Returns null. This sensor does not provide extended debugging information through this interface. |

## Integration Patterns

### Standard Usage
This component is not meant to be invoked directly. It is designed to be part of a collection of sensors evaluated by a higher-level AI system, such as a behavior selector or state transition manager. The system iterates through its sensors, calling `matches` on each to determine which behaviors are valid in the current game tick.

```java
// Hypothetical usage within a higher-level AI component

// The SensorMotionController is pre-configured and stored in a list
List<SensorBase> sensors = role.getActiveBehavior().getSensors();

// The AI system evaluates all sensors to make a decision
boolean canExecute = true;
for (SensorBase sensor : sensors) {
    if (!sensor.matches(entityRef, role, deltaTime, entityStore)) {
        canExecute = false;
        break;
    }
}

if (canExecute) {
    // ... proceed with behavior execution
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SensorMotionController()`. The object is invalid without being configured through its builder, which sets the required `motionControllerName`.
- **Stateful Misuse:** Do not treat this object as a mutable state container. It is a stateless evaluator. Any attempt to modify its internal fields via reflection will lead to undefined behavior across all NPCs using that shared behavior definition.
- **External Invocation:** Do not call the `matches` method from systems outside the core NPC update loop. The parameters it requires are deeply tied to the state of the game at a specific point in the tick cycle and are not safe to use with stale or synthetic data.

## Data Pipeline
The SensorMotionController acts as a conditional gate in the NPC decision-making pipeline. It does not transform data; it either permits or blocks the flow of logic based on the NPC's motion state.

> Flow:
> NPC Update Tick -> Behavior Logic Evaluation -> **SensorMotionController.matches(Role)** -> [TRUE] -> Behavior Execution / [FALSE] -> Behavior Rejection

