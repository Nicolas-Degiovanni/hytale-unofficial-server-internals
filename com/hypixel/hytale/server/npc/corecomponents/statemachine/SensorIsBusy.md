---
description: Architectural reference for SensorIsBusy
---

# SensorIsBusy

**Package:** com.hypixel.hytale.server.npc.corecomponents.statememachine
**Type:** Component / Transient

## Definition
```java
// Signature
public class SensorIsBusy extends SensorBase {
```

## Architecture & Concepts
SensorIsBusy is a highly specialized component within the server-side NPC Artificial Intelligence framework. It functions as a behavioral *guard* or *precondition* within an NPC's state machine. Its sole responsibility is to determine if an NPC is currently engaged in an uninterruptible action, formally known as a "busy state".

This class embodies the Single Responsibility Principle. It provides a clean, declarative way for AI designers to build transition logic without embedding state-checking code directly into higher-level behaviors. For example, a state transition that would cause an NPC to flee from danger should first check this sensor. If the NPC is busy (e.g., in the middle of a special attack animation), the transition will be blocked, preventing animation clipping and illogical behavior.

It is a leaf node in the AI's decision-making process, providing a simple boolean evaluation that feeds into more complex structures like Behavior Trees or Finite State Machines.

### Lifecycle & Ownership
- **Creation:** Instances are constructed during the server's NPC definition loading phase. They are not created per-tick or per-NPC instance, but rather as part of a shared, immutable AI behavior template. Instantiation is managed by a builder, typically `BuilderSensorBase`, which is itself populated from server configuration files (e.g., YAML or JSON).
- **Scope:** The object's lifetime is tied to the NPC's AI behavior definition. It is a stateless configuration object that persists as long as the corresponding NPC type is loaded in memory.
- **Destruction:** The object is marked for garbage collection when its parent AI definition is unloaded, for instance, when a world zone is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** **Stateless and Immutable.** A SensorIsBusy instance contains no mutable fields. Its behavior is determined entirely by the arguments passed to its `matches` method, particularly the `Role` object which holds the NPC's current state.
- **Thread Safety:** **Conditionally Thread-Safe.** The class itself is inherently thread-safe due to its stateless nature. However, its operational safety depends on the thread-safety of the objects passed into its methods. The NPC update loop is presumed to be single-threaded on a per-entity basis, making concurrent access within the intended architecture a non-issue.

**WARNING:** Invoking the `matches` method from an asynchronous task or a different thread without external locking on the `Role` and `EntityStore` objects will lead to race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates if the NPC is in a busy state. Returns true only if the base sensor conditions pass and the NPC's `StateSupport` confirms a busy status. |
| getSensorInfo() | InfoProvider | O(1) | Returns diagnostic information for the sensor. This implementation always returns null. |

## Integration Patterns

### Standard Usage
This component is not intended to be invoked directly in procedural code. It is designed to be declaratively integrated into an NPC's AI definition, which is then processed by the state machine engine.

The following conceptual example shows how it would be used within a hypothetical AI builder system.

```java
// During NPC AI definition loading
// This sensor is used as a condition to gate a state transition.
npcStateMachineBuilder
    .when(new SensorIsBusy(sensorBuilder))
    .then(new ActionContinueCurrentTask());

npcStateMachineBuilder
    .whenNot(new SensorIsBusy(sensorBuilder))
    .and(new SensorSeesEnemy(sensorBuilder))
    .then(new ActionEngageTarget());
```

### Anti-Patterns (Do NOT do this)
- **Ad-Hoc Checks:** Do not instantiate and use this sensor for one-off state checks within an action's update logic. This violates the declarative nature of the AI system. If you need to check for a busy state, query the `StateSupport` component directly.
- **Subclassing for Complexity:** Do not extend this class to add additional checks. If more complex logic is required (e.g., "is busy OR has low health"), compose multiple sensors together at the AI definition level using logical operators (AND, OR, NOT).

## Data Pipeline
SensorIsBusy acts as a gate in a control flow rather than a stage in a data processing pipeline. Its evaluation is a critical step in the NPC's main AI update tick.

> Flow:
> NPC AI Tick -> State Machine Engine -> Evaluate Transitions -> **SensorIsBusy.matches(role)** -> Boolean Result -> Allow / Deny State Change

