---
description: Architectural reference for SensorState
---

# SensorState

**Package:** com.hypixel.hytale.server.npc.corecomponents.statemachine
**Type:** Component

## Definition
```java
// Signature
public class SensorState extends SensorBase {
```

## Architecture & Concepts
The SensorState class is a concrete implementation of the SensorBase, designed to function as a conditional predicate within an NPC's AI State Machine. Its sole purpose is to evaluate whether an NPC entity is currently in a specific, predefined state. This makes it a fundamental component for creating robust, state-driven behaviors and controlling the flow of logic in the AI.

For example, a state transition from *Wandering* to *Attacking* might require a sensor that first confirms the NPC is not in a *Sleeping* or *Stunned* state. SensorState provides this exact capability.

A key architectural distinction within this class is its ability to operate in two modes, controlled by the `componentLocal` flag:
1.  **Global State Check:** When `componentLocal` is false, the sensor checks the primary state of the entire NPC `Role`. This is the most common use case, evaluating conditions like "Is the NPC in the Fleeing state?".
2.  **Component-Local State Check:** When `componentLocal` is true, the sensor checks the state of a *specific component* attached to the NPC. This allows for highly granular and encapsulated logic, where an individual component (e.g., a weapon manager) can have its own internal state machine that influences the broader AI.

## Lifecycle & Ownership
-   **Creation:** SensorState instances are **never** instantiated directly via code using the `new` keyword. They are exclusively constructed by the server's asset loading pipeline from NPC definition files (e.g., JSON assets). A `BuilderSensorState` object parses the asset and, with context from a `BuilderSupport` instance, creates the final, immutable SensorState object.
-   **Scope:** An instance of SensorState is a stateless, shared component. It is part of an NPC's static definition template. Its lifetime is tied to the parent NPC asset, persisting in memory as long as that NPC type is loaded by the server.
-   **Destruction:** Instances are garbage collected when the server unloads the corresponding NPC asset definitions. This typically occurs during a full server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields (`state`, `subState`, `componentLocal`, etc.) are `final` and are set only once during construction. An instance of this class represents a static, unchangeable condition definition.
-   **Thread Safety:** **Thread-safe**. Due to its immutable nature, a SensorState object can be safely accessed and evaluated by multiple threads without any need for external locking or synchronization. The `matches` method is a pure function relative to the object's own state; its outcome depends solely on the immutable configuration and the mutable `Role` object passed as an argument.

## API Surface
The public API is minimal and focused entirely on evaluation and diagnostics.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | The core evaluation method. Returns true if the provided Role's state matches the sensor's configured state. This is the primary entry point for the State Machine system. |
| getInfo(role, holder) | void | O(1) | Populates a diagnostic data structure with human-readable information about the state being checked. Used for debugging tools and server logs. |

## Integration Patterns

### Standard Usage
This component is not intended for direct, imperative use by developers. It is declaratively configured within an NPC asset file and consumed automatically by the server's core AI systems. The State Machine governing an NPC will hold a collection of these sensors and invoke their `matches` method during the AI update tick to determine if a state transition is warranted.

```java
// CONCEPTUAL: How the core State Machine system uses this sensor.
// This logic resides within the engine, not in typical gameplay code.

boolean canTransition = false;
for (SensorBase sensor : transition.getSensors()) {
    // The engine invokes matches() on each sensor, including SensorState instances.
    if (sensor.matches(entityRef, npcRole, deltaTime, entityStore)) {
        canTransition = true;
        break;
    }
}

if (canTransition) {
    // The State Machine proceeds with the state change.
    npcRole.getStateSupport().changeState(newState);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SensorState()`. The constructor requires resolved indices and context from the asset pipeline (`BuilderSupport`). Manual instantiation will result in an incorrectly configured sensor that will fail to function or cause unpredictable AI behavior due to invalid state or component indices.
-   **State Caching:** Do not cache the result of a `matches()` call. An NPC's state is volatile and can change on any tick. The sensor must be evaluated every time a condition needs to be checked to ensure the AI is reacting to the most current game state.

## Data Pipeline
SensorState acts as a conditional gate in the AI data flow. It does not transform data but instead produces a boolean signal that directs the flow of an NPC's logic.

> Flow:
> NPC Asset Definition (JSON) -> Server Asset Pipeline -> **SensorState Instance** (In-memory as part of a State Machine) -> AI Update Tick -> `matches(role)` Evaluation -> Boolean Result -> State Machine Transition Logic

